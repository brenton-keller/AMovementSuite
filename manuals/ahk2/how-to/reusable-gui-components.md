# How-To: Build Reusable GUI Components

## Problem

You're duplicating the same GUI patterns across multiple windows: labeled inputs, button panels, form sections, validation logic. Copy-pasting code makes maintenance a nightmare—fix a bug once, and you need to update it everywhere. Standard controls lack domain-specific behavior you need repeatedly.

## Quick Context

AHK2's class system enables sophisticated component architectures through **class extension** and **composition**. Unlike v1, every GUI and control is now a proper object you can extend. The key patterns:

1. **Extend `Gui` class** for custom dialogs with built-in behavior
2. **Method chaining** by returning `this` from builder methods
3. **Component wrappers** to bundle controls into reusable units
4. **Factory patterns** for consistent component creation
5. **Event sink pattern** using `this` as third parameter to `Gui.__New()`

For architectural understanding, see:
- docs/manuals/ahk2/explanation/gui-architecture.md
- docs/manuals/ahk2/explanation/oop-patterns.md

## Key Concepts

- **Event Sink**: Passing `this` to `super.__New(options, title, this)` makes your class instance the event sink, allowing method names as event handler strings
- **Method Chaining**: Return `this` from methods to enable fluent APIs: `obj.Method1().Method2().Method3()`
- **Composition**: Build complex components from simple ones rather than deep inheritance
- **Bind()**: Preserve context when passing methods as callbacks: `btn.OnEvent("Click", this.Handler.Bind(this))`

## Solution 1: Extend Gui Class for Custom Dialogs

**Use when**: You need a specialized dialog type used repeatedly (input, confirmation, settings).

```ahk
#Requires AutoHotkey v2.0

; Custom input dialog that returns a value
class InputDialog extends Gui {
    __New(options := "", title := "Input Required") {
        ; CRITICAL: Pass 'this' as third parameter for event sink pattern
        super.__New(options " +AlwaysOnTop", title, this)

        this.result := ""  ; Store user input
        this.CreateControls()
    }

    CreateControls() {
        this.SetFont("s10", "Segoe UI")

        ; Dynamic controls stored as properties
        this.promptText := this.Add("Text", "w300", "")
        this.inputEdit := this.Add("Edit", "w300")
        this.okBtn := this.Add("Button", "Default w100", "OK")
        this.cancelBtn := this.Add("Button", "x+10 w100", "Cancel")

        ; Event handlers as METHOD NAMES (event sink pattern)
        this.okBtn.OnEvent("Click", "OnOK")
        this.cancelBtn.OnEvent("Click", "OnCancel")

        return this  ; Enable chaining
    }

    ; Event handlers are now class methods
    OnOK(ctrl, info) {
        ; 'this' correctly refers to InputDialog instance
        this.result := this.inputEdit.Value
        this.Hide()
    }

    OnCancel(ctrl, info) {
        this.result := ""
        this.Hide()
    }

    ; Convenience method to show and retrieve input
    ShowDialog(prompt, defaultValue := "") {
        this.promptText.Text := prompt
        this.inputEdit.Value := defaultValue
        this.inputEdit.Focus()
        this.Show()

        ; Wait for dialog to close
        WinWaitClose("ahk_id " this.Hwnd)
        return this.result
    }
}

; Usage - simple and clean
dialog := InputDialog()

username := dialog.ShowDialog("Enter your username:", "Guest")
if username
    MsgBox("Welcome, " username "!")

email := dialog.ShowDialog("Enter your email:")
if email
    MsgBox("Email set to: " email)
```

**Why this works**: The event sink pattern (`this` as third param) means event handler strings like `"OnOK"` automatically resolve to methods on your class instance. Without this, you'd need to manually bind every handler.

## Solution 2: Fluent API with Method Chaining

**Use when**: Building dialogs programmatically with a readable, declarative syntax.

```ahk
#Requires AutoHotkey v2.0

class FluentDialog extends Gui {
    __New(title := "Dialog") {
        super.__New("+AlwaysOnTop -MinimizeBox", title, this)
        this.controls := []
        this.callbacks := Map()
    }

    ; All builder methods return 'this' for chaining

    WithMessage(text, options := "w300") {
        ctrl := this.Add("Text", "xm " options, text)
        this.controls.Push(ctrl)
        return this  ; ESSENTIAL for chaining
    }

    WithInput(label, varName := "", options := "w200") {
        this.Add("Text", "xm", label . ":")
        ctrl := this.Add("Edit", "x+10 " options " v" varName)
        this.controls.Push(ctrl)
        return this  ; ESSENTIAL for chaining
    }

    WithCheckbox(label, varName := "") {
        ctrl := this.Add("Checkbox", "xm v" varName, label)
        this.controls.Push(ctrl)
        return this  ; ESSENTIAL for chaining
    }

    WithDropdown(label, items, varName := "") {
        this.Add("Text", "xm", label . ":")
        ctrl := this.Add("DropDownList", "x+10 w200 v" varName, items)
        this.controls.Push(ctrl)
        return this  ; ESSENTIAL for chaining
    }

    WithButton(text, callback, isDefault := false) {
        opts := "xm w100"
        if isDefault
            opts .= " Default"

        btn := this.Add("Button", opts, text)
        if callback
            btn.OnEvent("Click", callback)

        return this  ; ESSENTIAL for chaining
    }

    WithSize(width, height) {
        this.width := width
        this.height := height
        return this  ; ESSENTIAL for chaining
    }

    Build() {
        w := this.HasProp("width") ? this.width : 400
        h := this.HasProp("height") ? this.height : 300
        this.Show("w" w " h" h)
        return this
    }
}

; Usage - reads like natural language!
registrationDialog := FluentDialog("User Registration")
    .WithSize(450, 350)
    .WithMessage("Please enter your registration details:")
    .WithInput("Full Name", "fullName", "w300")
    .WithInput("Email Address", "email", "w300")
    .WithInput("Password", "password", "w300 Password")
    .WithDropdown("Country", ["USA", "Canada", "UK", "Other"], "country")
    .WithCheckbox("I agree to the terms and conditions", "agreeTerms")
    .WithButton("Register", ProcessRegistration, true)
    .WithButton("Cancel", (*) => ExitApp)
    .Build()

ProcessRegistration(*) {
    submitted := registrationDialog.Submit(false)

    if !submitted.fullName {
        MsgBox("Name is required!", "Error", 16)
        return
    }
    if !submitted.email || !InStr(submitted.email, "@") {
        MsgBox("Valid email required!", "Error", 16)
        return
    }
    if !submitted.agreeTerms {
        MsgBox("You must agree to the terms!", "Error", 16)
        return
    }

    MsgBox("Registration submitted!`nName: " submitted.fullName
        . "`nEmail: " submitted.email
        . "`nCountry: " submitted.country,
        "Success", 64)
}
```

**Key insight**: Every builder method ends with `return this`, enabling the chain. This pattern creates highly readable code that's almost self-documenting.

## Solution 3: Component Wrapper Classes

**Use when**: You have UI patterns that always appear together (labeled input, button groups, validated fields).

```ahk
#Requires AutoHotkey v2.0

; Reusable labeled input component with built-in validation
class LabeledInput {
    __New(parentGui, label, options := "w200") {
        this.gui := parentGui
        this.validator := ""
        this.errorMsg := ""

        ; Create label and input as a unit
        this.labelCtrl := parentGui.Add("Text", "xm w100", label . ":")
        this.editCtrl := parentGui.Add("Edit", "x+10 " options)
    }

    ; Fluent validation configuration
    WithValidator(validatorFunc, errorMessage := "Invalid input") {
        this.validator := validatorFunc
        this.errorMsg := errorMessage
        return this
    }

    WithDefault(defaultValue) {
        this.editCtrl.Value := defaultValue
        return this
    }

    ; Property for easy value access
    Value {
        get => this.editCtrl.Value
        set => this.editCtrl.Value := value
    }

    ; Validation method
    IsValid() {
        if !this.validator
            return true
        return this.validator(this.Value)
    }

    GetErrorMessage() => this.errorMsg

    SetEnabled(enabled) {
        this.editCtrl.Enabled := enabled
        this.labelCtrl.Enabled := enabled
        return this
    }

    Focus() {
        this.editCtrl.Focus()
        return this
    }
}

; Reusable button panel component
class ButtonPanel {
    __New(parentGui, width := 0) {
        this.gui := parentGui
        this.buttons := []

        ; Add separator
        opts := "xm h2 +0x10"
        if width > 0
            opts .= " w" width
        parentGui.Add("Text", opts)
    }

    AddButton(text, callback, isDefault := false) {
        opts := "xm w100 h32"
        if isDefault
            opts .= " Default"

        btn := this.gui.Add("Button", opts, text)
        if callback
            btn.OnEvent("Click", callback)

        this.buttons.Push(btn)
        return this  ; Chain multiple AddButton calls
    }

    EnableAll() {
        for btn in this.buttons
            btn.Enabled := true
        return this
    }

    DisableAll() {
        for btn in this.buttons
            btn.Enabled := false
        return this
    }
}

; Usage - build a form with reusable components
myGui := Gui("+Resize", "User Profile")

; Create validated inputs
nameInput := LabeledInput(myGui, "Full Name", "w300")
    .WithValidator((v) => StrLen(Trim(v)) > 0, "Name is required")
    .WithDefault("John Doe")

emailInput := LabeledInput(myGui, "Email", "w300")
    .WithValidator((v) => InStr(v, "@") && InStr(v, "."), "Valid email required")

ageInput := LabeledInput(myGui, "Age", "w100 Number")
    .WithValidator((v) => IsNumber(v) && v >= 18 && v <= 120, "Age must be 18-120")

phoneInput := LabeledInput(myGui, "Phone", "w200")
    .WithValidator((v) => StrLen(RegExReplace(v, "\D")) >= 10, "Phone must have 10+ digits")

; Create button panel
buttonPanel := ButtonPanel(myGui, 400)
    .AddButton("Save", SaveProfile, true)
    .AddButton("Reset", ResetProfile)
    .AddButton("Cancel", (*) => ExitApp)

myGui.Show("w500")

SaveProfile(*) {
    ; Validate all fields
    fields := [nameInput, emailInput, ageInput, phoneInput]

    for field in fields {
        if !field.IsValid() {
            MsgBox(field.GetErrorMessage(), "Validation Error", 16)
            field.Focus()
            return
        }
    }

    ; All valid - save
    MsgBox("Profile saved!`n`nName: " nameInput.Value
        . "`nEmail: " emailInput.Value
        . "`nAge: " ageInput.Value
        . "`nPhone: " phoneInput.Value,
        "Success", 64)
}

ResetProfile(*) {
    nameInput.Value := ""
    emailInput.Value := ""
    ageInput.Value := ""
    phoneInput.Value := ""
}
```

**Benefits**:
- Validation logic lives with the component (DRY principle)
- Consistent appearance across your application
- Easy to extend with new methods (password strength, formatting, etc.)
- Single point of maintenance

## Solution 4: Component Composition Architecture

**Use when**: Building complex UIs from simpler building blocks (sections, panels, groups).

```ahk
#Requires AutoHotkey v2.0

; Base component class
class UIComponent {
    __New(parent) {
        this.parent := parent
        this.controls := []
    }

    AddControl(ctrl) {
        this.controls.Push(ctrl)
        return this
    }

    Enable() {
        for ctrl in this.controls
            if ctrl.HasProp("Enabled")
                ctrl.Enabled := true
        return this
    }

    Disable() {
        for ctrl in this.controls
            if ctrl.HasProp("Enabled")
                ctrl.Enabled := false
        return this
    }

    Show() {
        for ctrl in this.controls
            if ctrl.HasProp("Visible")
                ctrl.Visible := true
        return this
    }

    Hide() {
        for ctrl in this.controls
            if ctrl.HasProp("Visible")
                ctrl.Visible := false
        return this
    }
}

; Form section component
class FormSection extends UIComponent {
    __New(parent, title, width := 400) {
        super.__New(parent)
        this.width := width

        ; Section header
        parent.SetFont("s11 Bold")
        this.titleCtrl := parent.Add("Text", "xm w" width " h25 +0x200", title)
        parent.SetFont("s10")

        ; Separator line
        this.separator := parent.Add("Text", "xm w" width " h2 +0x10")

        this.AddControl(this.titleCtrl)
        this.AddControl(this.separator)
    }

    AddField(label, type := "Edit", options := "w200") {
        this.parent.Add("Text", "xm w100", label . ":")
        ctrl := this.parent.Add(type, "x+10 " options)
        this.AddControl(ctrl)
        return ctrl
    }

    AddCheckbox(label) {
        ctrl := this.parent.Add("Checkbox", "xm", label)
        this.AddControl(ctrl)
        return ctrl
    }

    AddButton(text, callback) {
        btn := this.parent.Add("Button", "xm w150", text)
        if callback
            btn.OnEvent("Click", callback)
        this.AddControl(btn)
        return btn
    }

    AddSpacer(height := 10) {
        this.parent.Add("Text", "xm h" height)
        return this
    }
}

; Tab section component
class TabSection extends UIComponent {
    __New(parent, tabNames*) {
        super.__New(parent)

        this.tabCtrl := parent.Add("Tab3", "xm w600 h400", tabNames)
        this.AddControl(this.tabCtrl)
        this.sections := Map()
    }

    GetSection(tabName) {
        if !this.sections.Has(tabName) {
            this.tabCtrl.UseTab(tabName)
            section := FormSection(this.parent, "", 580)
            this.sections[tabName] := section
        }
        return this.sections[tabName]
    }

    UseTab(tabName) {
        this.tabCtrl.UseTab(tabName)
        return this
    }

    EndTabs() {
        this.tabCtrl.UseTab()
        return this
    }
}

; Complete form builder
class FormBuilder extends Gui {
    __New(title) {
        super.__New("+Resize", title, this)
        this.components := []
    }

    AddSection(title, width := 400) {
        section := FormSection(this, title, width)
        this.components.Push(section)
        return section
    }

    AddTabs(tabNames*) {
        tabs := TabSection(this, tabNames*)
        this.components.Push(tabs)
        return tabs
    }

    Build() {
        this.Show("w620")
        return this
    }
}

; Usage - compose complex forms from simple components
form := FormBuilder("Employee Registration")

; Personal information section
personalSection := form.AddSection("Personal Information", 600)
personalSection.AddField("First Name", "Edit", "w250")
personalSection.AddField("Last Name", "Edit", "w250")
personalSection.AddField("Date of Birth", "Edit", "w150")
personalSection.AddCheckbox("US Citizen")
personalSection.AddSpacer(15)

; Employment section
employmentSection := form.AddSection("Employment Details", 600)
employmentSection.AddField("Position", "DropDownList", "w250")
    .Add(["Developer", "Designer", "Manager", "Analyst"])
employmentSection.AddField("Department", "DropDownList", "w250")
    .Add(["Engineering", "Design", "Sales", "HR"])
employmentSection.AddField("Start Date", "Edit", "w150")
employmentSection.AddField("Salary", "Edit", "w150")
employmentSection.AddSpacer(15)

; Benefits section with tabs
benefitsTabs := form.AddTabs("Health", "Retirement", "Other")

; Health tab
healthSection := benefitsTabs.GetSection("Health")
healthSection.AddCheckbox("Medical Insurance")
healthSection.AddCheckbox("Dental Insurance")
healthSection.AddCheckbox("Vision Insurance")

; Retirement tab
retirementSection := benefitsTabs.GetSection("Retirement")
retirementSection.AddCheckbox("401(k) Enrollment")
retirementSection.AddField("Contribution %", "Edit", "w100")

; Other tab
otherSection := benefitsTabs.GetSection("Other")
otherSection.AddCheckbox("Gym Membership")
otherSection.AddCheckbox("Transit Pass")

benefitsTabs.EndTabs()

; Buttons below tabs
form.Add("Text", "xm w600 h2 +0x10")
submitBtn := form.Add("Button", "xm w150 h35 Default", "Submit Registration")
cancelBtn := form.Add("Button", "x+10 w150 h35", "Cancel")

submitBtn.OnEvent("Click", (*) => MsgBox("Registration submitted!", "Success", 64))
cancelBtn.OnEvent("Click", (*) => ExitApp)

form.Build()
```

**What this demonstrates**:
- Building complex UIs from simple, reusable pieces
- Components can contain other components (composition)
- Each component manages its own controls
- Enable/Disable/Show/Hide entire sections at once
- Clean, maintainable code structure

## Solution 5: Factory Pattern for Consistent Components

**Use when**: You want to enforce consistent styling and behavior across all component instances.

```ahk
#Requires AutoHotkey v2.0

class ComponentFactory {
    ; Standard button with consistent styling
    static CreateButton(gui, text, callback := "", style := "primary") {
        opts := "h35"

        ; Style variants
        if style = "primary"
            opts .= " w150 Default"
        else if style = "secondary"
            opts .= " w120"
        else if style = "danger"
            opts .= " w120"

        btn := gui.Add("Button", opts, text)

        if callback
            btn.OnEvent("Click", callback)

        return btn
    }

    ; Labeled input with consistent layout
    static CreateLabeledInput(gui, label, varName := "", width := 200) {
        gui.Add("Text", "xm w120", label . ":")
        edit := gui.Add("Edit", "x+10 w" width " v" varName)
        return edit
    }

    ; Validated input with error display
    static CreateValidatedInput(gui, label, validator, errorMsg) {
        gui.Add("Text", "xm w120", label . ":")
        edit := gui.Add("Edit", "x+10 w200")
        errorText := gui.Add("Text", "x+10 w200 cRed Hidden", errorMsg)

        ; Return object with both controls and validator
        return {
            edit: edit,
            error: errorText,
            validator: validator,
            Validate: () => validator(edit.Value),
            ShowError: () => errorText.Visible := true,
            HideError: () => errorText.Visible := false,
            Value: edit.Value
        }
    }

    ; Checkbox group with consistent spacing
    static CreateCheckboxGroup(gui, title, options*) {
        controls := []

        gui.SetFont("s10 Bold")
        title := gui.Add("Text", "xm w300", title)
        gui.SetFont("s10")
        controls.Push(title)

        for option in options {
            cb := gui.Add("Checkbox", "xm", option)
            controls.Push(cb)
        }

        return controls
    }

    ; Button panel with standard layout
    static CreateButtonPanel(gui, buttons*) {
        gui.Add("Text", "xm w400 h2 +0x10")

        createdButtons := []
        for btnDef in buttons {
            style := btnDef.HasProp("style") ? btnDef.style : "secondary"
            btn := ComponentFactory.CreateButton(gui, btnDef.text,
                btnDef.HasProp("callback") ? btnDef.callback : "", style)
            createdButtons.Push(btn)
        }

        return createdButtons
    }

    ; Section header with consistent styling
    static CreateSectionHeader(gui, text, width := 400) {
        gui.SetFont("s12 Bold")
        header := gui.Add("Text", "xm w" width " h30 +0x200", text)
        gui.SetFont("s10")

        separator := gui.Add("Text", "xm w" width " h2 +0x10")

        return {header: header, separator: separator}
    }
}

; Usage - consistent components across the application
myGui := Gui("+Resize", "Factory Pattern Example")

ComponentFactory.CreateSectionHeader(myGui, "User Information", 450)

usernameEdit := ComponentFactory.CreateLabeledInput(myGui, "Username", "username", 250)
emailEdit := ComponentFactory.CreateLabeledInput(myGui, "Email", "email", 250)

ComponentFactory.CreateSectionHeader(myGui, "Preferences", 450)

checkboxes := ComponentFactory.CreateCheckboxGroup(myGui, "Notifications",
    "Email notifications",
    "Desktop alerts",
    "Mobile push"
)

ComponentFactory.CreateButtonPanel(myGui,
    {text: "Save", callback: SaveData, style: "primary"},
    {text: "Reset", callback: ResetData, style: "secondary"},
    {text: "Cancel", callback: (*) => ExitApp, style: "danger"}
)

myGui.Show("w500")

SaveData(*) {
    submitted := myGui.Submit(false)
    MsgBox("Data saved!`nUsername: " submitted.username "`nEmail: " submitted.email)
}

ResetData(*) {
    usernameEdit.Value := ""
    emailEdit.Value := ""
    for cb in checkboxes
        if cb.Type = "CheckBox"
            cb.Value := 0
}
```

**Benefits**:
- Consistent appearance and behavior
- Single location to update styles
- Enforces standards across team/project
- Easy to add new variants (themes, sizes)

## Complete Working Example: Reusable Settings Panel

```ahk
#Requires AutoHotkey v2.0
#SingleInstance Force

; =============================================================================
; REUSABLE COMPONENT SYSTEM - Complete Example
; Demonstrates: All component patterns in production-ready code
; =============================================================================

; ========== COMPONENT LIBRARY ==========

class ValidatedField {
    __New(parent, label, options := "w250") {
        this.parent := parent
        this.validator := ""
        this.errorMsg := ""

        parent.Add("Text", "xm w120", label . ":")
        this.edit := parent.Add("Edit", options " x+10")
        this.error := parent.Add("Text", "xp y+2 w250 cRed Hidden", "")
    }

    WithValidator(func, msg) {
        this.validator := func
        this.errorMsg := msg
        return this
    }

    Validate() {
        isValid := !this.validator || this.validator(this.edit.Value)
        this.error.Visible := !isValid
        this.error.Text := isValid ? "" : this.errorMsg
        return isValid
    }

    Value {
        get => this.edit.Value
        set => this.edit.Value := value
    }
}

class SettingsSection extends UIComponent {
    __New(parent, title) {
        super.__New(parent)
        this.fields := Map()

        parent.SetFont("s11 Bold")
        this.title := parent.Add("Text", "xm w500 h30 +0x200", title)
        parent.SetFont("s10")
        this.sep := parent.Add("Text", "xm w500 h2 +0x10")

        this.AddControl(this.title).AddControl(this.sep)
    }

    AddValidatedField(name, label, validator := "", errorMsg := "") {
        field := ValidatedField(this.parent, label)
        if validator
            field.WithValidator(validator, errorMsg)

        this.fields[name] := field
        this.AddControl(field.edit).AddControl(field.error)
        return field
    }

    AddCheckbox(name, label) {
        cb := this.parent.Add("Checkbox", "xm", label)
        this.fields[name] := cb
        this.AddControl(cb)
        return cb
    }

    ValidateAll() {
        for name, field in this.fields {
            if field.HasProp("Validate") && !field.Validate()
                return false
        }
        return true
    }

    GetValues() {
        values := Map()
        for name, field in this.fields {
            if field.HasProp("Value")
                values[name] := field.Value
        }
        return values
    }

    SetValues(values) {
        for name, value in values {
            if this.fields.Has(name) && this.fields[name].HasProp("Value")
                this.fields[name].Value := value
        }
    }
}

; Base class for component system
class UIComponent {
    __New(parent) {
        this.parent := parent
        this.controls := []
    }

    AddControl(ctrl) {
        this.controls.Push(ctrl)
        return this
    }

    Enable() {
        for ctrl in this.controls
            if ctrl.HasProp("Enabled")
                ctrl.Enabled := true
        return this
    }

    Disable() {
        for ctrl in this.controls
            if ctrl.HasProp("Enabled")
                ctrl.Enabled := false
        return this
    }
}

; ========== APPLICATION ==========

class SettingsApp extends Gui {
    __New() {
        super.__New("+Resize +MinSize500x400", "Application Settings", this)
        this.sections := Map()
        this.CreateUI()
        this.LoadSettings()
        this.Show("w550 h500")
    }

    CreateUI() {
        this.SetFont("s10", "Segoe UI")

        ; Account section
        account := SettingsSection(this, "Account Information")
        account.AddValidatedField("username", "Username",
            (v) => StrLen(Trim(v)) > 0,
            "Username cannot be empty")
        account.AddValidatedField("email", "Email",
            (v) => InStr(v, "@") && InStr(v, "."),
            "Invalid email address")
        this.sections["account"] := account

        ; Appearance section
        appearance := SettingsSection(this, "Appearance")
        appearance.AddCheckbox("darkMode", "Enable dark mode")
        appearance.AddCheckbox("animations", "Enable animations")
        this.sections["appearance"] := appearance

        ; Notifications section
        notifications := SettingsSection(this, "Notifications")
        notifications.AddCheckbox("emailNotif", "Email notifications")
        notifications.AddCheckbox("desktopNotif", "Desktop notifications")
        notifications.AddCheckbox("sounds", "Notification sounds")
        this.sections["notifications"] := notifications

        ; Buttons
        this.Add("Text", "xm w500 h2 +0x10")
        this.btnSave := this.Add("Button", "xm w120 h35 Default", "Save")
        this.btnReset := this.Add("Button", "x+10 w120 h35", "Reset")
        this.btnCancel := this.Add("Button", "x+10 w120 h35", "Cancel")

        this.btnSave.OnEvent("Click", (*) => this.SaveSettings())
        this.btnReset.OnEvent("Click", (*) => this.ResetSettings())
        this.btnCancel.OnEvent("Click", (*) => ExitApp)
    }

    LoadSettings() {
        ; Simulate loading from config file
        this.sections["account"].SetValues(Map(
            "username", "JohnDoe",
            "email", "john@example.com"
        ))
    }

    SaveSettings() {
        ; Validate all sections
        for name, section in this.sections {
            if section.HasProp("ValidateAll") && !section.ValidateAll() {
                MsgBox("Please correct the errors before saving.", "Validation Error", 16)
                return
            }
        }

        ; Collect all values
        allSettings := Map()
        for name, section in this.sections {
            if section.HasProp("GetValues") {
                for key, value in section.GetValues()
                    allSettings[key] := value
            }
        }

        ; In production: write to INI or registry
        msg := "Settings saved!`n`n"
        for key, value in allSettings
            msg .= key ": " value "`n"

        MsgBox(msg, "Success", 64)
    }

    ResetSettings() {
        result := MsgBox("Reset all settings to defaults?", "Confirm", "YesNo Icon?")
        if result = "Yes" {
            ; Reset all sections
            for name, section in this.sections
                section.SetValues(Map())
            this.LoadSettings()
        }
    }
}

; Launch application
app := SettingsApp()

; Development hotkey
^R:: Reload
```

## Troubleshooting

### Problem: Event handlers lose context (`this` is undefined)
**Solution**: Always use `.Bind(this)` when passing methods as callbacks:
```ahk
btn.OnEvent("Click", this.Handler.Bind(this))
```

OR use event sink pattern by passing `this` to `super.__New()`:
```ahk
super.__New(options, title, this)
btn.OnEvent("Click", "Handler")  ; Handler is a method name
```

### Problem: Method chaining breaks
**Solution**: Ensure EVERY builder method ends with `return this`:
```ahk
WithOption(value) {
    this.option := value
    return this  ; DON'T FORGET THIS
}
```

### Problem: Circular references prevent garbage collection
**Solution**: Explicitly destroy GUI objects and break references:
```ahk
this.gui.Destroy()
this.gui := ""
```

### Problem: Can't access parent GUI from component
**Solution**: Store parent reference in component:
```ahk
class MyComponent {
    __New(parent) {
        this.parent := parent  ; Store reference
        this.ctrl := parent.Add("Edit", "w200")
    }
}
```

### Problem: Components interfere with each other's positioning
**Solution**: Use consistent positioning (`xm` for margin, `x+10` for relative) or calculate absolute positions.

## Best Practices

1. **Event sink pattern**: Use `super.__New(options, title, this)` for cleaner event handlers
2. **Always return this**: Enable method chaining in all builder methods
3. **Composition over inheritance**: Build complex components from simple ones
4. **Store parent reference**: Components need access to parent GUI
5. **Validate in components**: Keep validation logic with the field
6. **Use factories for consistency**: Enforce standards across your app
7. **Document component APIs**: What methods, properties, events are available?
8. **Test components in isolation**: Create test GUIs for each component
9. **Version your components**: Track changes as you improve them
10. **Share components**: Build a library for your organization

## When to Use Each Pattern

| Pattern | Best For | Complexity | Reusability |
|---------|----------|------------|-------------|
| **Extend Gui** | Custom dialog types | Low | High |
| **Fluent API** | Programmatic dialog building | Medium | Medium |
| **Component Wrappers** | Repeated UI patterns | Low | Very High |
| **Composition** | Complex, hierarchical UIs | High | High |
| **Factory** | Consistent styling/behavior | Low | Very High |

## Further Reading

- docs/manuals/ahk2/explanation/oop-patterns.md - OOP design patterns in AHK2
- docs/manuals/ahk2/reference/gui-class.md - Gui class reference
- docs/manuals/ahk2/how-to/gui-mvc-pattern.md - MVC architecture
- docs/manuals/ahk2/explanation/event-handling.md - Deep dive into event system

## Related Community Resources

- **AHK v2 GUI Megathread**: https://www.autohotkey.com/boards/viewtopic.php?t=123220
- **Easy AutoGUI Designer**: https://github.com/samfisherirl/Easy-Auto-GUI-for-AHK-v2
