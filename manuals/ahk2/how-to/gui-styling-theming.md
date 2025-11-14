# How-To: Style and Theme Your GUI

## Problem

Default Windows controls look dated and unprofessional. You need custom colors, fonts, dark mode support, or branded styling. Your application needs to match corporate guidelines or modern design standards. Users expect polished interfaces with consistent theming.

## Quick Context

AHK2 provides **basic styling** through GUI options (`BackColor`, `SetFont`, color codes) and **advanced styling** through Windows API calls and custom drawing libraries. Unlike web frameworks or WPF, there's no CSS or XAML—styling requires a mix of native options and creative API usage.

**Three styling tiers**:
1. **Basic**: BackColor, SetFont, control color options (5 minutes to implement)
2. **Intermediate**: Dark mode title bars, ListView coloring (library-based)
3. **Advanced**: Custom controls with GDI+ graphics (requires external library)

For architectural concepts, see:
- docs/manuals/ahk2/explanation/gui-architecture.md
- docs/manuals/ahk2/reference/gui-options.md

## Key Concepts

- **BackColor**: Sets GUI window background color (hex or color name)
- **SetFont()**: Applies font settings to subsequently added controls
- **Control Color Options**: `c` prefix for text color, `+Background` for control backgrounds
- **DwmSetWindowAttribute**: Windows API for dark mode title bars
- **LV_Colors**: Community library for ListView styling
- **GpGFX**: Community library for custom graphics (gradients, shapes, images)

## Solution 1: Basic Colors and Fonts

**Use when**: You need simple color/font customization without external dependencies.

```ahk
#Requires AutoHotkey v2.0

; Basic styled GUI with custom colors and fonts
MyGui := Gui("+Resize", "Styled Application")

; Set window background color
MyGui.BackColor := "0xF0F0F0"  ; Light gray

; Set default font for all subsequent controls
MyGui.SetFont("s10", "Segoe UI")

; Header with custom font and color
MyGui.SetFont("s14 Bold", "Arial")
MyGui.Add("Text", "w400 cBlue", "Welcome to the Application")

; Reset to normal font
MyGui.SetFont("s10", "Segoe UI")

; Text with different colors
MyGui.Add("Text", "xm w400 cRed", "This text is red")
MyGui.Add("Text", "xm w400 c0x008800", "This text is green (hex color)")
MyGui.Add("Text", "xm w400", "Default text color")

; Edit with custom background
MyGui.Add("Edit", "xm w400 h100 +Background0xFFFFCC", "Yellow background edit box")

; Colored buttons (background color support is limited)
MyGui.SetFont("s10 Bold")
MyGui.Add("Button", "xm w150 h40 +Background0x0066CC", "Blue Button")
MyGui.Add("Button", "xm w150 h40 +Background0x00AA00", "Green Button")

MyGui.Show("w450")
```

**Color formats**:
- **Named**: `"White"`, `"Black"`, `"Red"`, `"Blue"`
- **Hex RGB**: `"0xRRGGBB"` (0xFF0000 = red, 0x00FF00 = green, 0x0000FF = blue)
- **Text color**: `c` prefix (e.g., `"cRed"` or `"c0xFF0000"`)

**Font options**:
- **Size**: `s10` (10 points)
- **Weight**: `Bold`, `Normal`
- **Style**: `Italic`, `Underline`, `Strike`
- **Quality**: `q4` (ClearType), `q5` (ClearType natural)

## Solution 2: Dark Mode GUI

**Use when**: Modern dark-themed interface is required (Windows 10 1809+).

```ahk
#Requires AutoHotkey v2.0

; Complete dark mode GUI with modern appearance
MyGui := Gui("+Resize", "Dark Mode Application")

; Dark background
MyGui.BackColor := "0x202020"

; Light text for dark background
MyGui.SetFont("s10 cWhite", "Segoe UI")

; Enable dark mode title bar (Windows 10/11)
EnableDarkMode(MyGui)

; Header
MyGui.SetFont("s14 Bold cWhite")
MyGui.Add("Text", "xm w500", "Dark Mode Interface")
MyGui.SetFont("s10 cWhite")

; Content sections
MyGui.Add("Text", "xm w500 h2 +0x10")  ; Separator

MyGui.Add("Text", "xm", "Username:")
MyGui.Add("Edit", "x+10 w300 +Background0x303030 cWhite")

MyGui.Add("Text", "xm", "Password:")
MyGui.Add("Edit", "x+10 w300 Password +Background0x303030 cWhite")

; Dark edit box for content
MyGui.Add("Edit", "xm w500 h200 +Multi +Background0x252525 cWhite",
    "Multi-line content area with dark background and light text.")

; Buttons
MyGui.Add("Button", "xm w120 h35", "Login")
MyGui.Add("Button", "x+10 w120 h35", "Cancel")

MyGui.Show("w550 h400")

; Function to enable dark mode title bar
EnableDarkMode(guiObj) {
    ; Check Windows version (requires 10.0.17763+)
    if (VerCompare(A_OSVersion, "10.0.17763") >= 0) {
        ; DWMWA_USE_IMMERSIVE_DARK_MODE attribute
        DWMWA_USE_IMMERSIVE_DARK_MODE := 19

        ; Windows 10 build 18985+ uses attribute 20
        if (VerCompare(A_OSVersion, "10.0.18985") >= 0)
            DWMWA_USE_IMMERSIVE_DARK_MODE := 20

        ; Apply dark mode to title bar
        DllCall("dwmapi\DwmSetWindowAttribute",
                "Ptr", guiObj.Hwnd,
                "Int", DWMWA_USE_IMMERSIVE_DARK_MODE,
                "Int*", true,
                "Int", 4)
    }
}
```

**Dark mode best practices**:
- **Background**: Use dark grays (0x202020 to 0x303030), not pure black
- **Text**: Use pure white or light gray (0xE0E0E0)
- **Contrast**: Ensure 4.5:1 minimum contrast ratio for readability
- **Accents**: Use saturated colors for buttons/highlights (0x0078D4 = Windows blue)
- **Test both modes**: Consider system theme detection with `RegRead`

**Detecting system dark mode**:
```ahk
IsSystemDarkMode() {
    try {
        val := RegRead("HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Themes\Personalize",
                       "AppsUseLightTheme")
        return !val  ; 0 = dark mode, 1 = light mode
    }
    return false
}
```

## Solution 3: ListView Styling with LV_Colors

**Use when**: You need colored rows, alternating colors, or cell-specific formatting in ListViews.

First, download **LV_Colors** from: https://www.autohotkey.com/boards/viewtopic.php?f=83&t=93922

```ahk
#Requires AutoHotkey v2.0
#Include <LV_Colors>  ; Place in Lib folder

MyGui := Gui("+Resize", "Styled ListView")
MyGui.BackColor := "0xF0F0F0"

; Create ListView
LV := MyGui.Add("ListView", "w500 h300", ["Name", "Status", "Priority", "Value"])

; Add sample data
LV.Add("", "Task 1", "Complete", "High", "100")
LV.Add("", "Task 2", "In Progress", "Medium", "50")
LV.Add("", "Task 3", "Pending", "Low", "25")
LV.Add("", "Task 4", "Complete", "High", "200")
LV.Add("", "Task 5", "Failed", "High", "0")
LV.Add("", "Task 6", "In Progress", "Medium", "75")

; Initialize LV_Colors
LVC := LV_Colors(LV)

; Alternating row colors (zebra striping)
LVC.AlternateRows(0xF0F8FF, 0xFFFFFF)  ; Light blue / White

; Color specific rows based on status
Loop LV.GetCount() {
    status := LV.GetText(A_Index, 2)

    if (status = "Complete")
        LVC.Row(A_Index, 0xD4EDDA, 0x155724)  ; Light green bg, dark green text
    else if (status = "Failed")
        LVC.Row(A_Index, 0xF8D7DA, 0x721C24)  ; Light red bg, dark red text
    else if (status = "In Progress")
        LVC.Row(A_Index, 0xFFF3CD, 0x856404)  ; Light yellow bg, dark yellow text
}

; Color specific columns
LVC.Col(4, , 0x0000FF)  ; Blue text for "Value" column

; Header colors
LVC.Header(0x333333, 0xFFFFFF)  ; Dark background, white text

MyGui.Show("w550")
```

**LV_Colors methods**:
- **AlternateRows(color1, color2)**: Zebra striping
- **Row(row, bgColor [, textColor])**: Color specific row
- **Cell(row, col, bgColor [, textColor])**: Color specific cell
- **Col(col, bgColor [, textColor])**: Color entire column
- **Header(bgColor [, textColor])**: Color header row
- **SelectionColors(bgColor, textColor)**: Selected row colors

**Color coding patterns**:
```ahk
; Status-based coloring
switch status {
    case "Success": LVC.Row(row, 0xD4EDDA, 0x155724)
    case "Warning": LVC.Row(row, 0xFFF3CD, 0x856404)
    case "Error":   LVC.Row(row, 0xF8D7DA, 0x721C24)
    case "Info":    LVC.Row(row, 0xD1ECF1, 0x0C5460)
}

; Value-based coloring (heatmap)
if (value > 100)
    LVC.Cell(row, col, 0xFF0000)  ; Red for high values
else if (value > 50)
    LVC.Cell(row, col, 0xFFFF00)  ; Yellow for medium
else
    LVC.Cell(row, col, 0x00FF00)  ; Green for low
```

## Solution 4: Custom Graphics with GpGFX

**Use when**: You need custom shapes, gradients, or completely custom-drawn controls.

Download **GpGFX** from: https://github.com/bceenaeiklmr/GpGFX

```ahk
#Requires AutoHotkey v2.0
#Include <GpGFX>  ; Place in Lib folder

; Create transparent overlay window
layer := Layer(, , 600, 400)

; Draw gradient background rectangle
background := Rectangle(0, 0, 600, 400)
background.Color := ["#1E3A8A", "#3B82F6"]  ; Dark blue to light blue gradient
background.Angle := 45  ; Gradient angle

; Draw header with text
header := Rectangle(0, 0, 600, 80)
header.Color := ["#1F2937", "#374151"]
header.str := "Custom Graphics Demo"
header.Font := "Segoe UI"
header.FontSize := 24
header.FontColor := "#FFFFFF"

; Draw rounded button
button := Rectangle(200, 150, 200, 50)
button.Color := ["#10B981", "#059669"]  ; Green gradient
button.Radius := 25  ; Rounded corners
button.str := "Click Me"
button.FontSize := 16
button.FontColor := "#FFFFFF"

; Draw circle
circle := Ellipse(450, 250, 100, 100)
circle.Color := "#EF4444"  ; Red
circle.Border := "#991B1B"  ; Dark red border
circle.BorderWidth := 3

; Draw custom shape (triangle)
triangle := Polygon([[100, 300], [150, 200], [200, 300]])
triangle.Color := "#FBBF24"  ; Yellow

; Render all graphics
Draw(layer)

; Keep window open
Sleep(5000)
End()  ; Clean up GDI+ resources
```

**GpGFX features**:
- **Shapes**: Rectangle, Ellipse, Polygon, Line, Bezier curves
- **Gradients**: Linear gradients with angle control
- **Text**: Centered text with custom fonts
- **Images**: Load PNG/JPG with filters (sepia, grayscale, blur)
- **Transparency**: Alpha channel support
- **Layering**: Multiple layers for complex compositions

**Creating custom buttons with GpGFX**:
```ahk
class CustomButton {
    __New(layer, x, y, width, height, text) {
        this.rect := Rectangle(x, y, width, height)
        this.rect.Color := ["#3B82F6", "#1E40AF"]
        this.rect.Radius := 10
        this.rect.str := text
        this.rect.FontColor := "#FFFFFF"
        this.rect.FontSize := 14
        this.isHovered := false
    }

    OnHover() {
        this.rect.Color := ["#60A5FA", "#2563EB"]  ; Lighter gradient
        Draw(layer)
    }

    OnNormal() {
        this.rect.Color := ["#3B82F6", "#1E40AF"]  ; Original gradient
        Draw(layer)
    }
}
```

## Solution 5: Modern Web-Based UI with WebViewToo

**Use when**: You need fully modern, CSS-styleable interfaces with rich interactivity.

Download **WebViewToo** from: https://github.com/The-CoDingman/WebViewToo

```ahk
#Requires AutoHotkey v2.0
#Include <WebViewToo>  ; Requires Edge WebView2 runtime

; Create modern web-based GUI
wv := WebViewToo()

; Load HTML content with CSS styling
html := "
(
<!DOCTYPE html>
<html>
<head>
    <style>
        * {
            margin: 0;
            padding: 0;
            box-sizing: border-box;
        }

        body {
            font-family: 'Segoe UI', system-ui, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            height: 100vh;
            display: flex;
            align-items: center;
            justify-content: center;
        }

        .container {
            background: white;
            padding: 40px;
            border-radius: 20px;
            box-shadow: 0 20px 60px rgba(0,0,0,0.3);
            max-width: 400px;
            width: 90%;
        }

        h1 {
            color: #333;
            margin-bottom: 20px;
            font-size: 28px;
        }

        .input-group {
            margin-bottom: 20px;
        }

        label {
            display: block;
            color: #555;
            margin-bottom: 8px;
            font-weight: 500;
        }

        input {
            width: 100%;
            padding: 12px 16px;
            border: 2px solid #e1e8ed;
            border-radius: 8px;
            font-size: 16px;
            transition: border-color 0.3s;
        }

        input:focus {
            outline: none;
            border-color: #667eea;
        }

        .btn {
            width: 100%;
            padding: 14px;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            border: none;
            border-radius: 8px;
            font-size: 16px;
            font-weight: 600;
            cursor: pointer;
            transition: transform 0.2s;
        }

        .btn:hover {
            transform: translateY(-2px);
        }

        .btn:active {
            transform: translateY(0);
        }
    </style>
</head>
<body>
    <div class='container'>
        <h1>Modern Login</h1>
        <div class='input-group'>
            <label for='username'>Username</label>
            <input type='text' id='username' placeholder='Enter username'>
        </div>
        <div class='input-group'>
            <label for='password'>Password</label>
            <input type='password' id='password' placeholder='Enter password'>
        </div>
        <button class='btn' onclick='login()'>Sign In</button>
    </div>

    <script>
        function login() {
            const username = document.getElementById('username').value;
            const password = document.getElementById('password').value;

            // Send data to AHK
            window.chrome.webview.postMessage({
                action: 'login',
                username: username,
                password: password
            });
        }
    </script>
</body>
</html>
)"

wv.LoadHTML(html)
wv.Show("w800 h600 Center")

; Receive messages from web page
wv.OnWebMessage := WebMessageReceived

WebMessageReceived(wv, data) {
    ; Parse JSON message from JavaScript
    if (data.action = "login") {
        MsgBox("Login attempt:`nUsername: " data.username "`nPassword: " data.password,
               "Login", 64)
    }
}
```

**WebViewToo advantages**:
- **Modern CSS**: Flexbox, Grid, animations, transitions
- **JavaScript libraries**: Use React, Vue, jQuery, etc.
- **Responsive**: Mobile-first design patterns
- **Rich components**: Charts, calendars, rich text editors
- **No styling limitations**: Full control over appearance

## Complete Working Example: Themed Settings Panel

```ahk
#Requires AutoHotkey v2.0
#SingleInstance Force

; =============================================================================
; THEMED SETTINGS PANEL - Complete Example
; Demonstrates: Dark mode, custom colors, modern styling
; =============================================================================

class ThemedSettings extends Gui {
    __New(useDarkMode := true) {
        super.__New("+Resize +MinSize500x400", "Application Settings", this)

        this.darkMode := useDarkMode
        this.ApplyTheme()
        this.CreateControls()
        this.OnEvent("Size", this.HandleResize.Bind(this))
        this.Show("w600 h550")
    }

    ApplyTheme() {
        if this.darkMode {
            ; Dark theme
            this.bgColor := "0x1E1E1E"
            this.textColor := "White"
            this.editBg := "0x2D2D30"
            this.accentColor := "0x0078D4"

            this.BackColor := this.bgColor
            this.SetFont("s10 c" this.textColor, "Segoe UI")
            EnableDarkMode(this)
        } else {
            ; Light theme
            this.bgColor := "0xF3F3F3"
            this.textColor := "Black"
            this.editBg := "0xFFFFFF"
            this.accentColor := "0x0066CC"

            this.BackColor := this.bgColor
            this.SetFont("s10 c" this.textColor, "Segoe UI")
        }
    }

    CreateControls() {
        ; Header
        this.SetFont("s16 Bold")
        this.header := this.Add("Text", "xm ym w580 h50 Center +0x200", "Settings")
        this.SetFont("s10")

        ; Theme toggle
        this.CreateSection("Appearance")
        this.themeToggle := this.Add("Checkbox", "xm", "Dark Mode")
        this.themeToggle.Value := this.darkMode
        this.themeToggle.OnEvent("Click", (*) => this.ToggleTheme())

        ; Account section
        this.CreateSection("Account")
        this.Add("Text", "xm w120", "Username:")
        this.usernameEdit := this.Add("Edit", "x+10 w300 +Background" this.editBg)

        this.Add("Text", "xm w120", "Email:")
        this.emailEdit := this.Add("Edit", "x+10 w300 +Background" this.editBg)

        ; Preferences section
        this.CreateSection("Preferences")
        this.notifCheck := this.Add("Checkbox", "xm", "Enable notifications")
        this.soundCheck := this.Add("Checkbox", "xm", "Enable sounds")
        this.autoCheck := this.Add("Checkbox", "xm", "Auto-start with Windows")

        ; Status bar
        this.status := this.Add("Text", "xm w580 h30 Center Border +Background" this.editBg,
            "Ready")

        ; Buttons with custom colors
        this.btnSave := this.Add("Button", "xm w150 h40", "Save Settings")
        this.btnReset := this.Add("Button", "x+10 w150 h40", "Reset")
        this.btnCancel := this.Add("Button", "x+10 w150 h40", "Cancel")

        ; Event handlers
        this.btnSave.OnEvent("Click", (*) => this.Save())
        this.btnReset.OnEvent("Click", (*) => this.Reset())
        this.btnCancel.OnEvent("Click", (*) => ExitApp)
    }

    CreateSection(title) {
        this.SetFont("s11 Bold")
        this.Add("Text", "xm w580 h30 +0x200", title)
        this.SetFont("s10")
        this.Add("Text", "xm w580 h2 +0x10")
    }

    ToggleTheme() {
        ; Save current values
        username := this.usernameEdit.Value
        email := this.emailEdit.Value
        notif := this.notifCheck.Value
        sound := this.soundCheck.Value
        auto := this.autoCheck.Value

        ; Toggle theme
        this.darkMode := !this.darkMode

        ; Recreate GUI
        this.Destroy()
        newGui := ThemedSettings(this.darkMode)

        ; Restore values
        newGui.usernameEdit.Value := username
        newGui.emailEdit.Value := email
        newGui.notifCheck.Value := notif
        newGui.soundCheck.Value := sound
        newGui.autoCheck.Value := auto
    }

    Save() {
        this.status.Text := "Settings saved successfully!"
        this.status.Opt("c0x00AA00")  ; Green
        SetTimer(() => this.ResetStatus(), -2000)
    }

    Reset() {
        this.usernameEdit.Value := ""
        this.emailEdit.Value := ""
        this.notifCheck.Value := 0
        this.soundCheck.Value := 0
        this.autoCheck.Value := 0

        this.status.Text := "Settings reset to defaults"
        this.status.Opt("c0x0078D4")  ; Blue
        SetTimer(() => this.ResetStatus(), -2000)
    }

    ResetStatus() {
        this.status.Text := "Ready"
        this.status.Opt("c" this.textColor)
    }

    HandleResize(guiObj, MinMax, Width, Height) {
        if MinMax = -1
            return

        margin := 10
        availW := Width - (2 * margin)

        this.header.Move(margin, , availW)
        this.status.Move(margin, Height - 80, availW)
    }
}

EnableDarkMode(guiObj) {
    if (VerCompare(A_OSVersion, "10.0.17763") >= 0) {
        DWMWA_USE_IMMERSIVE_DARK_MODE := 19
        if (VerCompare(A_OSVersion, "10.0.18985") >= 0)
            DWMWA_USE_IMMERSIVE_DARK_MODE := 20

        DllCall("dwmapi\DwmSetWindowAttribute",
                "Ptr", guiObj.Hwnd,
                "Int", DWMWA_USE_IMMERSIVE_DARK_MODE,
                "Int*", true,
                "Int", 4)
    }
}

; Launch with dark mode
app := ThemedSettings(true)

; Development hotkey
^R:: Reload
```

## Troubleshooting

### Problem: Dark mode title bar doesn't work
**Solution**: Requires Windows 10 build 17763+. Check OS version:
```ahk
if (VerCompare(A_OSVersion, "10.0.17763") < 0)
    MsgBox("Dark mode title bar requires Windows 10 1809+")
```

### Problem: Button background colors don't apply
**Solution**: Windows buttons don't support `BackColor` well. Options:
1. Use owner-drawn buttons (complex)
2. Use GpGFX for custom buttons
3. Use WebViewToo for full CSS control
4. Accept default button styling

### Problem: Text is unreadable on colored backgrounds
**Solution**: Ensure sufficient contrast (4.5:1 minimum):
- Dark bg (0x202020) → Light text (0xFFFFFF)
- Light bg (0xF0F0F0) → Dark text (0x000000)

### Problem: Colors look different on different monitors
**Solution**: Use standard color spaces and test on multiple displays. Stick to web-safe colors or Windows system colors.

### Problem: LV_Colors doesn't update dynamically
**Solution**: After modifying ListView data, call:
```ahk
LVC.UpdateColors()  ; Refresh all colors
```

## Best Practices

1. **Contrast ratios**: Maintain 4.5:1 minimum for accessibility
2. **Consistent palette**: Define theme colors once, reuse throughout
3. **Test both themes**: Support light and dark modes
4. **Font hierarchy**: Use size and weight to establish visual hierarchy
5. **Limit colors**: Stick to 3-5 main colors plus neutrals
6. **System fonts**: Use "Segoe UI" on Windows for native look
7. **Responsive colors**: Ensure colors work at all window sizes
8. **Performance**: Avoid excessive redrawing with custom graphics
9. **Version check**: Verify OS version before using modern APIs
10. **User preference**: Let users choose theme when possible

## When to Use Each Approach

| Approach | Complexity | Capabilities | Performance |
|----------|------------|--------------|-------------|
| **Basic (BackColor/SetFont)** | Low | Limited colors/fonts | Excellent |
| **Dark Mode API** | Low | Modern title bar | Excellent |
| **LV_Colors** | Low | Rich ListView styling | Good |
| **GpGFX** | High | Custom graphics | Good |
| **WebViewToo** | Medium | Unlimited CSS styling | Fair |

## Further Reading

- docs/manuals/ahk2/reference/gui-options.md - All GUI style options
- docs/manuals/ahk2/reference/gui-methods.md - SetFont and other methods
- docs/manuals/ahk2/explanation/windows-api-integration.md - Using Windows APIs
- docs/manuals/ahk2/how-to/responsive-gui-layouts.md - Layout techniques

## Related Community Resources

- **LV_Colors Library**: https://www.autohotkey.com/boards/viewtopic.php?f=83&t=93922
- **GpGFX Library**: https://github.com/bceenaeiklmr/GpGFX
- **WebViewToo**: https://github.com/The-CoDingman/WebViewToo
- **AHK v2 GUI Megathread**: https://www.autohotkey.com/boards/viewtopic.php?t=123220
