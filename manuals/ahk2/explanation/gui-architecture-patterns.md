# GUI Architecture Patterns in AutoHotkey v2

## Purpose

This document explains **why** AHK2's GUI system is designed the way it is, **how** its core patterns work under the hood, and **what** architectural decisions shape professional GUI development. This is not a how-to guide—it's a deep dive into the philosophy, mechanisms, and mental models that make AHK2 GUI programming distinctive.

---

## 1. The Fundamental Shift: From Procedural to Object-Oriented

### Why AHK v1 → v2 Changed Everything

AHK v1 treated GUIs as procedural constructs. You called `Gui, Add, Button` and referenced controls through cryptic variable names or command sequences. Event handlers were G-labels—goto targets scattered throughout your script. The GUI existed as a collection of side effects rather than as a cohesive, queryable object.

**The architectural problem**: Procedural GUIs don't compose. You couldn't package a dialog as a reusable unit. Testing was impossible without manual interaction. State management meant tracking global variables. Extending behavior required copy-paste programming.

AHK v2 fundamentally reimagined GUIs as **first-class objects**. Every GUI is now an instance of the `Gui` class. Every control returned by `Gui.Add()` is an object with methods and properties. Event handlers are methods or function references. The entire interface is a structured object graph you can traverse, extend, and reason about programmatically.

**The architectural advantage**: Object-oriented GUIs enable:
- **Encapsulation**: A dialog class packages structure, behavior, and state together
- **Inheritance**: Extend `Gui` to create custom dialog types with specialized behavior
- **Composition**: Build complex interfaces from reusable component objects
- **Testing**: Mock GUI interactions by substituting test doubles
- **Type safety**: Properties enforce correct value types (where AHK's dynamic typing allows)

### What "GUI as Class Object" Means Architecturally

When you write `class MyDialog extends Gui`, you're not just organizing code—you're creating a **type** with:

1. **Constructor logic** (`__New`) that builds the interface deterministically
2. **Private state** (instance properties) that tracks dialog-specific data
3. **Public interface** (methods) that defines how external code interacts
4. **Event handlers** (methods) that implement behavior in response to user actions

```ahk
class LoginDialog extends Gui {
    __New() {
        super.__New("+AlwaysOnTop", "Login", this)  ; 'this' is event sink

        ; State encapsulated in instance
        this.authenticated := false
        this.attemptCount := 0

        ; Structure defined in constructor
        this.usernameEdit := this.Add("Edit", "w300")
        this.passwordEdit := this.Add("Edit", "w300 Password")
        this.loginBtn := this.Add("Button", "Default w100", "Login")

        ; Behavior wired to methods
        this.loginBtn.OnEvent("Click", "OnLogin")
    }

    ; Public interface
    Authenticate() {
        this.Show()
        ; Could block here or use callbacks for async result
        return this.authenticated
    }

    ; Private event handling (event sink pattern)
    OnLogin(ctrl, info) {
        this.attemptCount++
        ; Validate, update state, close window...
    }
}
```

**Key insight**: The GUI object's lifetime determines resource lifecycle. When the object is destroyed (no more references), Windows resources are freed. This is impossible with procedural code where cleanup is manual and error-prone.

### The Shift in Mental Model

| Aspect | v1 (Procedural) | v2 (Object-Oriented) |
|--------|-----------------|----------------------|
| GUI creation | `Gui, New` (command) | `myGui := Gui()` (constructor call) |
| Control addition | `Gui, Add, Button` (side effect) | `btn := myGui.Add("Button")` (returns object) |
| Event handling | `Gui, Submit` + G-labels | `btn.OnEvent("Click", handler)` (observer pattern) |
| State storage | Global variables | Instance properties |
| Reusability | Copy-paste code | Class inheritance/composition |
| Testing | Manual UI interaction | Object mocking |

**Why this matters**: Object-oriented design enables **separation of concerns**. Your dialog's business logic (validation, data processing) can live in separate classes. The GUI becomes a **view layer** that renders state and dispatches user actions to a controller. This is the foundation of MVC architecture (explained in section 6).

---

## 2. Why No Native Anchoring: Design Philosophy

### The Manual Layout Decision

Modern GUI frameworks (WinForms, WPF, Qt) provide **layout managers**—systems that automatically position controls based on declarative rules. You specify "anchor this button to the bottom-right" or "make this panel fill available space" and the framework handles the math.

AHK2 provides **none of this**. Every responsive layout requires manually writing an `OnSize` event handler that calls `Control.Move()` with calculated positions.

**Why would AHK2 omit such a fundamental feature?**

Three architectural reasons:

1. **Minimalism**: AHK2 is a *wrapper* around Win32 API, not a comprehensive framework. Adding a layout engine would mean shipping a constraint solver, flex-box logic, or grid system—significant complexity that most AHK users wouldn't need for simple automation UIs.

2. **Transparency**: Manual layouts are explicit. When you read `EditBox.Move(, , Width - 20, Height - 60)`, you know exactly what's happening. Layout managers introduce **implicit behavior**—controls move for non-obvious reasons, and debugging requires understanding the layout algorithm.

3. **Control**: Advanced layouts (like pinning one corner while allowing asymmetric scaling) often require custom logic. Layout managers excel at common cases but become awkward for edge cases. AHK2's philosophy: give you full control, accept the verbosity.

### The OnSize + Move() Architecture

The architecture is simple:

```
User resizes window
    ↓
Windows sends WM_SIZE message
    ↓
AHK2 triggers OnSize event
    ↓
Your handler receives (GuiObj, MinMax, Width, Height)
    ↓
You calculate new positions/sizes
    ↓
You call Control.Move(X, Y, W, H) for each control
    ↓
Windows redraws controls
```

**Critical detail**: `Move()` parameters are **optional**. Omitted parameters preserve existing values. This enables:

```ahk
; Anchor to bottom (move Y, preserve X/Width/Height)
button.Move(, Height - 40)

; Expand width only (preserve X/Y/Height)
editBox.Move(, , Width - 20)

; Reposition and resize
panel.Move(10, 10, Width - 20, Height - 60)
```

The pattern is verbose but **compositional**—each control's positioning is independent, and complex layouts are just sequences of `Move()` calls.

### Control vs Convenience Tradeoff

**What you lose**: Declarative layout syntax, automatic constraint satisfaction, built-in common patterns (docking, anchoring, flow layout).

**What you gain**:
- **Zero abstraction overhead**: Your code directly calls Win32 `SetWindowPos`
- **Predictable behavior**: No hidden layout passes or constraint conflicts
- **Full flexibility**: Implement non-standard layouts (curved arrangements, physics-based, responsive to external data)
- **Small runtime**: No layout engine to load/parse/execute

**Why community libraries exist**: The community filled the gap. GuiReSizer provides percentage-based positioning—a middle ground between manual math and full layout engines. It demonstrates AHK2's extensibility: the core stays minimal, users add sophistication as needed.

---

## 3. How Method Chaining Works: The Fluent Interface Pattern

### The `return this` Architecture

Method chaining (fluent interfaces) enables code like:

```ahk
dialog := FluentDialog("Settings")
    .WithTitle("Application Settings")
    .WithInput("Username", "username")
    .WithInput("Email", "email")
    .WithButton("OK", SaveHandler)
    .Build()
```

Each method call returns the object itself, allowing the next method to be called on the same object. This is **syntactic sugar**—the methods could be called separately:

```ahk
dialog := FluentDialog("Settings")
dialog.WithTitle("Application Settings")
dialog.WithInput("Username", "username")
dialog.WithInput("Email", "email")
dialog.WithButton("OK", SaveHandler)
dialog.Build()
```

But chaining **reads like natural language** and reduces cognitive load.

### How It Works Under the Hood

Every builder method follows this pattern:

```ahk
class FluentDialog extends Gui {
    WithInput(label, varName) {
        ; Do the work (add controls, store state, etc.)
        this.Add("Text", "xm", label)
        ctrl := this.Add("Edit", "x+10 w200 v" varName)
        this.controls.Push(ctrl)

        ; Return self to enable chaining
        return this
    }
}
```

The key is **`return this`**. Without it, the method returns `unset`, and you can't chain:

```ahk
; Without 'return this':
dialog.WithInput("Name", "name")  ; Returns unset
    .WithInput("Email", "email")  ; ERROR: can't call method on unset
```

### Call Chain Visualization

```
dialog := FluentDialog()
    ↓ (returns FluentDialog instance)
.WithTitle("Settings")
    ↓ (executes method, returns 'this')
.WithInput("Username")
    ↓ (executes method, returns 'this')
.WithButton("OK")
    ↓ (executes method, returns 'this')
.Build()
    ↓ (executes method, shows GUI, returns 'this')

Result: All methods operate on same object instance
```

### Why It's Readable

Chaining groups related operations visually. Compare:

**Without chaining:**
```ahk
dialog := FluentDialog("Settings")
dialog.SetSize(400, 300)
dialog.AddField("Name", "name")
dialog.AddField("Email", "email")
dialog.AddButton("OK", handler)
dialog.Show()
```

**With chaining:**
```ahk
dialog := FluentDialog("Settings")
    .SetSize(400, 300)
    .AddField("Name", "name")
    .AddField("Email", "email")
    .AddButton("OK", handler)
    .Show()
```

The chained version communicates: "Configure this dialog by applying a sequence of transformations." The non-chained version reads like unrelated statements. Chaining makes the **builder pattern** explicit.

### When Not to Use Chaining

Fluent interfaces are appropriate for **configuration** (building an object state before use). They're inappropriate for:

- **Queries**: Methods that return data should return that data, not `this`
- **Side effects**: Operations that primarily affect external state (file I/O, network calls) shouldn't chain—it hides failure modes
- **Conditional logic**: Chaining assumes linear configuration; branching logic breaks the flow

Example of misuse:

```ahk
; BAD: Chaining query methods
result := database
    .Connect()
    .Query("SELECT * FROM users")  ; Should return query result, not database
    .Close()  ; Now we've lost the query result!
```

Chaining is a **design pattern** for specific use cases, not a universal principle.

---

## 4. Event Sink Pattern Explained: The `this` Parameter Mystery

### The Problem: Method Names as Strings

In object-oriented AHK2, event handlers are often **methods** of your GUI class:

```ahk
class MyDialog extends Gui {
    __New() {
        super.__New()
        btn := this.Add("Button", "w100", "Click Me")

        ; We want OnClick to be a method
        btn.OnEvent("Click", "OnClick")  ; String method name
    }

    OnClick(ctrl, info) {
        MsgBox("Button clicked!")
    }
}
```

**The mystery**: How does AHK2 know to call `OnClick` on **this instance** of `MyDialog`? The string `"OnClick"` doesn't reference any object. Where does the method lookup happen?

### How Event Dispatch Works

When you pass a **string** to `OnEvent()`, AHK2 doesn't immediately resolve it to a method. Instead, it stores the string and defers resolution until the event fires. But it needs a **context object** to search for the method.

That context object is the **event sink**.

### The Event Sink Parameter

The third parameter to `Gui.__New()` designates the event sink:

```ahk
class MyDialog extends Gui {
    __New() {
        super.__New(options, title, this)  ; ← 'this' is event sink

        btn := this.Add("Button", "w100", "OK")
        btn.OnEvent("Click", "OnOK")  ; String method name
    }

    OnOK(ctrl, info) {
        ; Called when button clicked
    }
}
```

**What happens when the button is clicked:**

1. Windows sends `WM_COMMAND` message to GUI window
2. AHK2 receives message, identifies it as button click event
3. AHK2 looks up registered handler for this button's Click event
4. Handler is stored as string `"OnOK"`
5. AHK2 retrieves the **event sink object** (the `MyDialog` instance)
6. AHK2 checks if event sink has a method/property named `"OnOK"`
7. If found, calls it: `eventSink.OnOK(ctrl, info)`

### Event Sink Resolution Diagram

```
┌──────────────────────────────────────────────────────┐
│ User clicks button                                    │
└───────────────────┬──────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────────────┐
│ Windows: Send WM_COMMAND to HWND                     │
└───────────────────┬──────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────────────┐
│ AHK2: Map HWND → GUI object                          │
│       Map control ID → control object                │
└───────────────────┬──────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────────────┐
│ Control object: Retrieve OnEvent("Click") handlers   │
└───────────────────┬──────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────────────┐
│ Handler is string "OnOK"                             │
│   → Look up EventSink.OnOK                           │
│   → EventSink is 'this' passed to Gui.__New()       │
└───────────────────┬──────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────────────┐
│ Call EventSink.OnOK(controlObj, info)                │
└──────────────────────────────────────────────────────┘
```

### Why Bind() Is the Alternative

If you **don't** pass `this` as event sink (or can't use string method names), you must explicitly bind methods:

```ahk
class MyDialog extends Gui {
    __New() {
        super.__New()  ; No event sink specified

        btn := this.Add("Button", "w100", "OK")

        ; Must bind 'this' explicitly
        btn.OnEvent("Click", this.OnOK.Bind(this))
    }

    OnOK(ctrl, info) {
        ; 'this' refers to MyDialog instance because we bound it
    }
}
```

**What Bind() does**: Creates a new function object where the first parameter (`this` context) is pre-filled:

```
Original method:  OnOK(ctrl, info)
Bound function:   BoundOnOK(ctrl, info)  where 'this' = MyDialog instance
```

### Why Event Sink Pattern Exists

Two reasons:

1. **Convenience**: String method names are cleaner than `.Bind(this)` on every handler
2. **Dynamic dispatch**: You can override methods in subclasses, and the event system will call the overridden version—this is polymorphism

**Tradeoff**: Event sink adds indirection. If you mistype a method name string, you won't know until the event fires (runtime error). Bind() gives you earlier feedback (compile-time or first-access error).

---

## 5. Component Composition Architecture: Building Blocks Approach

### Why Composition Over Inheritance

Object-oriented design offers two ways to reuse code:

- **Inheritance**: "A LoginDialog *is a* Gui" (extends)
- **Composition**: "A LoginDialog *has a* Gui" (contains)

AHK2 GUI classes typically use **inheritance** (`extends Gui`) for convenience—you get all Gui methods automatically. But for **components** (reusable UI elements smaller than a full dialog), **composition** is superior.

**Why inheritance breaks down for components:**

Imagine you want a reusable "labeled input field" (text label + edit box). If you use inheritance:

```ahk
class LabeledInput extends Gui {
    __New(label) {
        super.__New()
        this.Add("Text", "xm", label)
        this.Add("Edit", "x+10 w200")
    }
}
```

**Problem**: Each `LabeledInput` is a **separate window**. You can't embed three labeled inputs in a single dialog window. Inheritance makes `LabeledInput` a top-level GUI, not a component.

### Composition Solution

Instead, components **receive** a parent GUI and add controls to it:

```ahk
class LabeledInput {
    __New(parentGui, label, options := "w200") {
        this.parent := parentGui

        ; Add controls to parent's window
        this.labelCtrl := parentGui.Add("Text", "xm", label . ":")
        this.editCtrl := parentGui.Add("Edit", "x+10 " options)
    }

    Value {
        get => this.editCtrl.Value
        set => this.editCtrl.Value := value
    }
}
```

Now you can compose multiple inputs in one dialog:

```ahk
myGui := Gui()
nameInput := LabeledInput(myGui, "Name", "w300")
emailInput := LabeledInput(myGui, "Email", "w300")
myGui.Show()
```

Each `LabeledInput` is a **logical grouping** of controls, not a separate window. This is composition.

### How Wrapper Classes Encapsulate Complexity

Components hide implementation details behind a simple interface:

```ahk
class LabeledInput {
    __New(parent, label) {
        this.label := parent.Add("Text", "xm w100", label)
        this.edit := parent.Add("Edit", "x+10 w200")
        this.errorText := parent.Add("Text", "x+5 w150 cRed Hidden", "")

        this.validator := ""
        this.required := false
    }

    WithValidator(func) {
        this.validator := func
        return this
    }

    WithRequired() {
        this.required := true
        return this
    }

    IsValid() {
        val := this.edit.Value

        if (this.required && !StrLen(val)) {
            this.errorText.Text := "Required"
            this.errorText.Visible := true
            return false
        }

        if (this.validator && !this.validator(val)) {
            this.errorText.Text := "Invalid"
            this.errorText.Visible := true
            return false
        }

        this.errorText.Visible := false
        return true
    }
}
```

**Encapsulation benefits:**

- **External code** just calls `input.IsValid()`—doesn't need to know about error labels
- **Implementation** can change (e.g., show errors as tooltips instead) without affecting usage
- **Reusability**: Copy `LabeledInput` to any project, it works identically

### Factory Pattern Purpose

Factories standardize component creation:

```ahk
class DialogFactory {
    static CreateFormField(gui, label, type := "Edit") {
        return LabeledInput(gui, label)
            .WithValidator(DefaultValidator)
            .WithRequired()
    }

    static CreateButtonBar(gui, ...buttons) {
        gui.Add("Text", "xm w500 h2 +0x10")  ; Separator

        for btn in buttons {
            ctrl := gui.Add("Button", "xm w100", btn.text)
            ctrl.OnEvent("Click", btn.handler)
            if (btn.default)
                ctrl.Opt("+Default")
        }
    }
}
```

**Why factories matter:**

1. **Consistency**: All form fields created through factory have same validation/styling
2. **Centralized configuration**: Change all button widths by editing factory
3. **Testability**: Mock the factory to inject test-specific components

### Component Hierarchy Example

```
SettingsDialog (Gui instance)
├─ PersonalSection (FormSection component)
│  ├─ NameInput (LabeledInput component)
│  │  ├─ Label (Text control)
│  │  ├─ Edit (Edit control)
│  │  └─ ErrorLabel (Text control)
│  ├─ EmailInput (LabeledInput component)
│  └─ PhoneInput (LabeledInput component)
├─ AccountSection (FormSection component)
│  ├─ UsernameInput
│  └─ PasswordInput
└─ ButtonBar (ButtonPanel component)
   ├─ SaveButton
   └─ CancelButton
```

Each component:
- **Owns** its child controls (manages their lifecycle)
- **Exposes** a clean interface (properties/methods)
- **Hides** implementation details (internal state, helper controls)
- **Composes** with other components (doesn't know about siblings)

This is **component-based architecture**—the same pattern used by React, Vue, and modern UI frameworks.

---

## 6. MVC in AHK Context: Why Separate Concerns

### The Problem: Monolithic GUI Classes

Without architecture, GUI code becomes a tangled mess:

```ahk
class MyApp extends Gui {
    __New() {
        super.__New()

        ; GUI structure
        this.nameEdit := this.Add("Edit", "w200")
        this.listView := this.Add("ListView", "w400 h300")
        this.saveBtn := this.Add("Button", "w100", "Save")

        ; Event handlers with business logic
        this.saveBtn.OnEvent("Click", (*) => this.Save())
    }

    Save() {
        ; Validation logic
        if (!StrLen(this.nameEdit.Value))
            return MsgBox("Name required!")

        ; Data processing
        name := StrUpper(this.nameEdit.Value)

        ; File I/O
        FileAppend(name "`n", "data.txt")

        ; UI update
        this.listView.Add("", name)
        this.nameEdit.Value := ""

        ; More UI logic
        MsgBox("Saved!")
    }
}
```

**Problems:**

1. **Can't test validation** without creating a GUI window
2. **Can't reuse data processing** in a console app
3. **Can't change file format** without editing GUI class
4. **Can't replace ListView with TreeView** without touching business logic

This is **tight coupling**—every component depends on every other component.

### MVC Separation

Model-View-Controller separates concerns:

- **Model**: Data structures and business logic (validation, processing, storage)
- **View**: GUI rendering and user input collection
- **Controller**: Coordinates Model and View, handles user actions

```
    ┌──────────────┐
    │  Controller  │  ← Entry point, orchestrates everything
    └──────┬───────┘
           │
      ┌────┴────┐
      ↓         ↓
  ┌───────┐  ┌──────┐
  │ Model │  │ View │
  └───────┘  └──────┘
  (data/logic) (GUI)
```

### Model → Controller → View Flow

```
User clicks "Save" button
    ↓
View.OnSaveClick() called
    ↓
View calls Controller.Save(inputValue)
    ↓
Controller validates using Model.Validate()
    ↓ (if valid)
Controller processes using Model.Process()
    ↓
Controller saves using Model.Store()
    ↓
Model triggers "data changed" event
    ↓
Controller receives event
    ↓
Controller calls View.Refresh(Model.GetAll())
    ↓
View updates ListView with new data
```

**Key insight**: Each layer only knows about **one direction**:

- View knows Controller exists (calls its methods)
- Controller knows both exist (calls both)
- Model knows **neither** exists (just fires events)

This is **dependency inversion**—high-level modules (View/Controller) depend on abstractions (Model's public interface), not details.

### Observer Pattern for View Updates

How does the View know when Model data changes? **Observer pattern** (also called publish-subscribe):

```ahk
class DataModel {
    items := []
    observers := []  ; List of callback functions

    Add(item) {
        this.items.Push(item)
        this.NotifyObservers()  ; Trigger all watchers
    }

    Subscribe(callback) {
        this.observers.Push(callback)
    }

    NotifyObservers() {
        for observer in this.observers
            observer(this.items)
    }
}

class Controller {
    __New() {
        this.model := DataModel()
        this.view := AppView(this)

        ; Subscribe to model changes
        this.model.Subscribe((items) => this.view.Refresh(items))
    }

    AddItem(text) {
        this.model.Add(text)  ; Model update automatically refreshes view
    }
}
```

**Flow:**

```
Controller.AddItem() → Model.Add() → Model.NotifyObservers()
                                              ↓
                            View.Refresh() ← callback fired
```

The View is **reactive**—it responds to Model changes automatically. The Controller doesn't manually call `view.Refresh()`; the observer pattern handles it.

### Why GUI Shouldn't Contain Business Logic

**Testability**: You can test business logic in isolation:

```ahk
; Test validation without GUI
validator := DataValidator()
assert(validator.IsValidEmail("test@example.com"))
assert(!validator.IsValidEmail("invalid"))
```

**Reusability**: Business logic can be used in:
- Console apps (no GUI)
- Web services (HTTP endpoints)
- Scheduled tasks (no user interaction)

**Maintainability**: Changing validation rules doesn't require touching 10 different event handlers.

**Replaceability**: Swap the GUI for a web interface without rewriting business logic.

---

## 7. State Management Patterns: Observable Properties

### The Problem: Manual UI Updates

Without state management, keeping UI in sync with data requires manual updates:

```ahk
class MyApp extends Gui {
    __New() {
        super.__New()
        this.username := "Guest"
        this.label := this.Add("Text", "w200", "User: Guest")
        this.changeBtn := this.Add("Button", "w200", "Change User")
        this.changeBtn.OnEvent("Click", (*) => this.ChangeUser())
    }

    ChangeUser() {
        this.username := "John"
        this.label.Text := "User: " this.username  ; Manual update

        ; If 5 controls display username, need 5 manual updates
    }
}
```

**Scaling problems:**

- Forget one update → UI shows stale data
- Username displayed in 10 places → 10 manual updates
- Username changes from multiple sources → complex update coordination

### How Reactive Updates Work

**Observable properties** automatically notify watchers when changed:

```ahk
class ObservableProperty {
    __New(initialValue := "") {
        this._value := initialValue
        this._observers := []
    }

    Value {
        get => this._value
        set {
            if (this._value != value) {
                this._value := value
                this.Notify()  ; Trigger all watchers
            }
        }
    }

    Subscribe(callback) {
        this._observers.Push(callback)
        return this
    }

    Notify() {
        for observer in this._observers
            observer(this._value)
    }
}
```

**Usage:**

```ahk
username := ObservableProperty("Guest")

myGui := Gui()
label1 := myGui.Add("Text", "w200", "User: Guest")
label2 := myGui.Add("Text", "w200", "Welcome, Guest")
title := myGui.Add("Text", "w200", "Guest's Dashboard")

; Subscribe all controls to username changes
username.Subscribe((val) => label1.Text := "User: " val)
username.Subscribe((val) => label2.Text := "Welcome, " val)
username.Subscribe((val) => title.Text := val "'s Dashboard")

myGui.Add("Button", "w200", "Change User")
    .OnEvent("Click", (*) => username.Value := "John")  ; One change, three updates

myGui.Show()
```

**When you set** `username.Value := "John"`, the property automatically:
1. Checks if value actually changed (prevents unnecessary updates)
2. Stores new value
3. Calls all subscribed callbacks with new value

Each callback updates its UI element. **One write, multiple reads**—this is reactive programming.

### Property Get/Set as Observer Triggers

AHK2 properties can run code on access:

```ahk
class User {
    __New(name) {
        this._name := name
        this._nameObservers := []
    }

    Name {
        get => this._name
        set {
            if (this._name != value) {
                this._name := value
                this.NotifyNameChanged()
            }
        }
    }

    SubscribeToName(callback) {
        this._nameObservers.Push(callback)
    }

    NotifyNameChanged() {
        for observer in this._nameObservers
            observer(this._name)
    }
}
```

**Usage:**

```ahk
user := User("Alice")

myGui := Gui()
nameLabel := myGui.Add("Text", "w200", user.Name)

user.SubscribeToName((name) => nameLabel.Text := name)

myGui.Add("Button", "w200", "Change Name")
    .OnEvent("Click", (*) => user.Name := "Bob")  ; Automatic UI update

myGui.Show()
```

**Key difference from plain properties**: The `set` accessor **intercepts writes**, enabling side effects (notifications) without changing calling code.

### Data Binding Architecture

Full data binding connects UI controls to model properties bidirectionally:

```
┌─────────────┐         ┌─────────────┐
│   Model     │◄────────┤    View     │
│  Property   │────────►│   Control   │
└─────────────┘         └─────────────┘
  (data source)          (UI element)

Model changes → Control updates
Control changes → Model updates
```

**Implementation pattern:**

```ahk
class Binding {
    __New(modelProperty, control, controlProperty := "Text") {
        this.model := modelProperty
        this.control := control
        this.controlProp := controlProperty

        ; Model → Control
        modelProperty.Subscribe((val) => this.UpdateControl(val))

        ; Control → Model (for editable controls)
        if (control.Type = "Edit")
            control.OnEvent("Change", (*) => this.UpdateModel())
    }

    UpdateControl(value) {
        this.control.%this.controlProp% := value
    }

    UpdateModel() {
        this.model.Value := this.control.%this.controlProp%
    }
}
```

**Usage:**

```ahk
username := ObservableProperty("Guest")

myGui := Gui()
editBox := myGui.Add("Edit", "w200")
label := myGui.Add("Text", "w200")

; Bind both controls to same model property
Binding(username, editBox, "Text")   ; Two-way (Edit syncs back to model)
Binding(username, label, "Text")      ; One-way (Text updates from model)

myGui.Show()

; Now: typing in editBox updates label automatically
; And: changing username.Value updates both controls
```

**Why this matters**: Data binding eliminates manual synchronization code. The architecture enforces **single source of truth** (the model property), and all views automatically reflect current state.

---

## 8. Why Community Libraries Exist: Filling Framework Gaps

### What AHK2 Intentionally Doesn't Provide

AHK2's GUI system is deliberately **minimal**:

❌ No layout managers (anchoring, docking, flexbox, grid)
❌ No data binding framework
❌ No built-in validation system
❌ No theming/styling engine beyond basic colors
❌ No advanced controls (color picker, date/time picker, rich text editor)
❌ No animation framework
❌ No built-in web browser control

**Why the minimalism?**

1. **Scope**: AHK2 is an automation language, not a GUI framework. The goal is "good enough for automation UIs," not "compete with WPF."

2. **Maintenance**: Complex frameworks require extensive maintenance. The core AHK2 team is small. Features that don't serve the majority add maintenance burden.

3. **Flexibility**: Minimal core means maximum flexibility. Adding a layout engine would impose design constraints. Manual layouts allow any custom behavior.

4. **Dependency-free**: AHK2 has zero dependencies. Advanced GUI features would require shipping libraries (e.g., a web engine).

5. **Community-driven**: The community can build specialized solutions faster than core team can generalize them. Libraries evolve independently without waiting for AHK2 releases.

### GuiReSizer: Addressing Percentage Positioning

**What it solves**: Manual percentage calculations in OnSize handlers.

**How it works**:

```ahk
#Include <GuiReSizer>

myGui := Gui("+Resize")
myGui.OnEvent("Size", GuiReSizer)  ; Use library's event handler

edit := myGui.Add("Edit", "w400 h300")
edit.WP := 0.90  ; Width = 90% of window
edit.HP := 0.80  ; Height = 80% of window

btn := myGui.Add("Button", "w100 h30", "OK")
btn.XP := 0.85   ; X position = 85% from left
btn.YP := 0.90   ; Y position = 90% from top
```

**Architecture**: GuiReSizer's `OnEvent("Size")` handler:
1. Receives new window dimensions
2. Iterates all controls
3. Reads custom properties (WP, HP, XP, YP)
4. Calculates pixel positions from percentages
5. Calls `Control.Move()` with calculated values

**Why it's not in core**: Percentage-based positioning is **one** layout strategy. Others might want:
- Anchor-based (WinForms style)
- Constraint-based (iOS Auto Layout)
- Flow-based (HTML flexbox)
- Grid-based (CSS Grid)

Core AHK2 can't include all these, so it includes **none** and lets community experiment.

### WebViewToo: Bringing Modern Web Tech

**What it solves**: Rich UI capabilities (HTML/CSS/JS) without reimplementing a rendering engine.

**How it works**:

```ahk
#Include <WebViewToo>

wv := WebViewToo()
wv.Load("
(
<!DOCTYPE html>
<html>
<head>
    <style>
        body { font-family: Arial; background: #f0f0f0; }
        .container { max-width: 600px; margin: 50px auto; padding: 20px;
                     background: white; border-radius: 8px; box-shadow: 0 2px 10px rgba(0,0,0,0.1); }
    </style>
</head>
<body>
    <div class='container'>
        <h1>Modern UI in AHK2</h1>
        <button onclick='sendToAHK()'>Click Me</button>
    </div>
    <script>
        function sendToAHK() {
            window.chrome.webview.postMessage('button-clicked');
        }
    </script>
</body>
</html>
)")

wv.OnMessage((data) => MsgBox("Received: " data))
wv.Show("w800 h600")
```

**Architecture**:
- Uses Microsoft Edge WebView2 (Chromium-based browser control)
- Embeds full web browser in AHK2 window
- Provides AHK↔JavaScript bridge for communication

**Why it's not in core**:
1. **Size**: WebView2 runtime is 100+ MB
2. **Dependency**: Requires Edge WebView2 installed (Windows 11 includes it, Windows 10 may not)
3. **Complexity**: Maintaining JavaScript bridge, handling async operations, managing web security
4. **Use case**: Only valuable for complex UIs; overkill for simple automation dialogs

**When to use**: Dashboard UIs, data visualization, rich text editing, embedding web content, leveraging web frameworks (React, Vue, etc.)

### The Library Ecosystem Philosophy

AHK2's approach mirrors **Unix philosophy**:

> Make each program do one thing well. To do a new job, build afresh rather than complicate old programs by adding new features.

**Core AHK2** does: Automation, scripting, basic GUIs
**Libraries** do: Advanced layouts, web UIs, custom graphics, UI automation, specialized controls

**Benefits**:
- **Core stays stable**: Few breaking changes between versions
- **Libraries evolve rapidly**: Experimental features don't destabilize core
- **User choice**: Pick only the libraries you need, avoid bloat
- **Community ownership**: Library authors maintain their code, core team doesn't bottleneck

**Drawbacks**:
- **Discovery**: Finding the right library requires research
- **Quality variation**: No official vetting of community libraries
- **Compatibility**: Libraries may lag behind core updates
- **Fragmentation**: Multiple libraries solving same problem differently

This tradeoff is **intentional**. AHK2 prioritizes **flexibility and stability** over **batteries-included convenience**.

---

## 9. Debunked Misconceptions

### Misconception 1: "AHK2 has layout managers like WinForms"

**Reality**: AHK2 has **zero** built-in layout management. Every responsive layout requires manually writing `OnSize` handlers with `Move()` calculations.

**Why the confusion**:
- GuiReSizer library provides percentage positioning, which resembles layout managers
- AHK v1 had `Gui, Add, ... AnchorLeft` through third-party libraries
- Other automation tools (AutoIt, PowerShell GUI) have some layout features

**The truth**: AHK2 deliberately omits layout managers. You must:
- Calculate positions/sizes manually based on window dimensions
- Use community libraries (GuiReSizer) for percentage-based positioning
- Or accept fixed-size GUIs (no `+Resize` option)

**Why it matters**: Don't expect declarative anchoring like `Anchor := "Right,Bottom"`. You're writing imperative positioning code.

---

### Misconception 2: "GUI variables can be local to functions"

**Reality**: GUIs **can** be local, but they'll be destroyed when the function exits (garbage collection), closing the window.

**The confusion**:

```ahk
CreateGUI() {
    local myGui := Gui()  ; Local variable
    myGui.Add("Text", "w200", "Hello")
    myGui.Show()
    ; Function ends, myGui goes out of scope
    ; → GUI object is destroyed
    ; → Window closes immediately
}

CreateGUI()  ; Window flashes and disappears
```

**Why**: AHK2's garbage collector destroys objects with no references. When `myGui` (the only reference) goes out of scope, the object is freed, calling its destructor, which closes the window.

**Solutions**:

1. **Make GUI global**:
```ahk
CreateGUI() {
    global myGui := Gui()  ; Persists after function
    myGui.Show()
}
```

2. **Store in persistent object**:
```ahk
class App {
    static gui := ""
    static Create() {
        App.gui := Gui()  ; Stored in static property
        App.gui.Show()
    }
}
App.Create()  ; Window persists
```

3. **Return GUI object**:
```ahk
CreateGUI() {
    myGui := Gui()
    myGui.Show()
    return myGui
}
global mainWindow := CreateGUI()  ; Caller keeps reference
```

**Why it matters**: Understanding object lifetime prevents mysterious "window closes immediately" bugs.

---

### Misconception 3: "Properties store values"

**Reality**: Properties are **computed accessors**. They run code on get/set; they don't inherently store anything.

**The confusion**:

```ahk
class Person {
    Name {
        get => StrUpper(this._name)  ; Computes value on each access
        set => this._name := value   ; Stores in backing field
    }
}
```

People assume `this.Name` is a **field** (storage location). Actually:
- `this.Name` **calls** the `get` accessor (runs code)
- `this.Name := "John"` **calls** the `set` accessor (runs code)
- The **actual storage** is `this._name` (a separate field)

**Why it matters**:

```ahk
class Counter {
    Count {
        get {
            Static c := 0
            return ++c  ; Different value each access!
        }
    }
}

counter := Counter()
MsgBox(counter.Count)  ; 1
MsgBox(counter.Count)  ; 2 (computed, not stored!)
```

Properties enable:
- **Computed values** (like combining first/last name)
- **Validation** (reject invalid values in setter)
- **Side effects** (trigger events when value changes)
- **Lazy loading** (calculate expensive value only when accessed)

**Don't confuse** properties with fields. Properties are **methods disguised as fields**.

---

### Misconception 4: "Event handlers don't need parameters"

**Reality**: Event handlers **must** accept parameters (at minimum 2 for most events), even if unused. Use `*` to ignore them.

**The error**:

```ahk
MyButton.OnEvent("Click", HandleClick)

HandleClick() {  ; Missing parameters!
    MsgBox("Clicked")
}
; → Runtime error when button clicked
```

**Why**: AHK2 calls event handlers with **mandatory parameters**:
- `OnEvent("Click")` passes `(GuiCtrlObj, Info)`
- `OnEvent("Size")` passes `(GuiObj, MinMax, Width, Height)`

If your function doesn't accept them, **parameter count mismatch error**.

**Solutions**:

```ahk
; Accept parameters explicitly
HandleClick(ctrl, info) {
    MsgBox("Clicked")
}

; OR use * to accept any parameters
HandleClick(*) {
    MsgBox("Clicked")
}

; OR use fat arrow syntax with *
MyButton.OnEvent("Click", (*) => MsgBox("Clicked"))
```

**Why it matters**: Missing parameters is the #1 beginner error with AHK2 events. Always use `(ctrl, info)` or `(*)`.

---

### Misconception 5: "Circular references in GUIs are safe"

**Reality**: Circular references **prevent garbage collection**, leaking memory. You must explicitly break cycles.

**The problem**:

```ahk
class MyApp {
    __New() {
        this.gui := Gui()
        this.gui.app := this  ; Circular reference: this.gui.app.gui.app...

        ; Both objects now have 2+ references
        ; When function exits, neither can be garbage collected
    }
}

CreateApp() {
    local app := MyApp()
    ; Function ends, but 'app' object persists (leaked)
}

Loop 1000
    CreateApp()  ; Creates 1000 leaked GUI objects
```

**Why circular refs leak**:

```
app object: references = 2 (local variable 'app' + gui.app property)
gui object: references = 2 (app.gui property + Windows HWND)

Function exits:
  - 'app' variable removed → app.references = 1 (still gui.app)
  - app.references > 0 → NOT garbage collected
  - gui.references > 0 → NOT garbage collected

Result: Both leaked
```

**Solutions**:

1. **Avoid circular references**:
```ahk
class MyApp {
    __New() {
        this.gui := Gui()
        ; Don't store 'this' in GUI
        this.gui.OnEvent("Close", this.OnClose.Bind(this))  ; Use Bind instead
    }
}
```

2. **Explicitly break cycle**:
```ahk
class MyApp {
    __New() {
        this.gui := Gui()
        this.gui.app := this
    }

    Destroy() {
        this.gui.app := ""  ; Break circular reference
        this.gui.Destroy()   ; Free Windows resources
    }
}

app := MyApp()
; When done:
app.Destroy()
```

**Why it matters**: Long-running scripts with many GUI creations/destructions will leak memory without proper cleanup.

---

### Misconception 6: "OnSize fires only when user resizes"

**Reality**: `OnSize` fires during **maximize/minimize/restore**, not just manual dragging.

**The confusion**: People test resizing by dragging window borders, and `OnSize` works. They assume it's only for manual resizes.

**When OnSize actually fires**:
- Manual resize (drag borders)
- Maximize button clicked
- Minimize button clicked
- Restore (from minimized/maximized)
- Programmatic size changes (`Gui.Move()`)
- Show() with size specified

**Critical detail**: The `MinMax` parameter indicates:
- `-1` = Minimized
- `0` = Restored/resized
- `1` = Maximized

**Why it matters**:

```ahk
Gui_Size(guiObj, MinMax, Width, Height) {
    ; BUG: Resizes controls when window minimized
    EditBox.Move(, , Width - 20, Height - 60)
}
```

When minimized, `Width` and `Height` are **0**, so this sets control size to **-20 × -60** (invalid). You get visual glitches or errors.

**Correct**:

```ahk
Gui_Size(guiObj, MinMax, Width, Height) {
    if (MinMax = -1)  ; Skip processing when minimized
        return

    EditBox.Move(, , Width - 20, Height - 60)
}
```

---

### Misconception 7: "Event sink and Bind() are interchangeable"

**Reality**: They achieve similar results but have different **scope** and **dispatch mechanisms**.

**Event sink** (passing `this` to `Gui.__New()`):

```ahk
class MyDialog extends Gui {
    __New() {
        super.__New(options, title, this)  ; Event sink
        btn := this.Add("Button", "w100", "OK")
        btn.OnEvent("Click", "OnOK")  ; String method name
    }

    OnOK(ctrl, info) {
        ; Called via event sink lookup
    }
}
```

**Bind()** (explicit binding):

```ahk
class MyDialog extends Gui {
    __New() {
        super.__New()  ; No event sink
        btn := this.Add("Button", "w100", "OK")
        btn.OnEvent("Click", this.OnOK.Bind(this))  ; Explicit bind
    }

    OnOK(ctrl, info) {
        ; Called via bound function object
    }
}
```

**Key differences**:

| Aspect | Event Sink | Bind() |
|--------|-----------|--------|
| Scope | All events for this GUI | Single event handler |
| Method names | Strings ("OnOK") | Direct references (this.OnOK) |
| Polymorphism | Subclasses can override | Bound to specific implementation |
| Error detection | Runtime (method not found) | Earlier (missing method) |
| Flexibility | GUI-wide convention | Per-handler choice |

**When to use event sink**: Building a cohesive GUI class where all events route through class methods.

**When to use Bind()**: One-off handlers, callbacks to external functions, need early error detection.

**Why it matters**: Understanding the mechanism prevents confusion when mixing both approaches (which is valid).

---

## 10. ASCII Diagrams

### OnSize Event Flow

```
┌────────────────────────────────────────────────────────────────┐
│                    User resizes window                          │
│              (drag border, maximize, restore)                   │
└─────────────────────────┬──────────────────────────────────────┘
                          ↓
┌────────────────────────────────────────────────────────────────┐
│                  Windows sends WM_SIZE message                  │
│                    to window handle (HWND)                      │
└─────────────────────────┬──────────────────────────────────────┘
                          ↓
┌────────────────────────────────────────────────────────────────┐
│         AHK2 message pump receives WM_SIZE                      │
│           Maps HWND → Gui object instance                       │
└─────────────────────────┬──────────────────────────────────────┘
                          ↓
┌────────────────────────────────────────────────────────────────┐
│       Gui object: Check for OnEvent("Size") handlers           │
└─────────────────────────┬──────────────────────────────────────┘
                          ↓
                    Handler exists?
                     ╱           ╲
                  YES              NO
                   ↓                ↓
┌──────────────────────────┐   ┌──────────────┐
│ Call handler with args:  │   │  Do nothing  │
│ - GuiObj (this GUI)      │   └──────────────┘
│ - MinMax (-1, 0, or 1)   │
│ - Width (client area)    │
│ - Height (client area)   │
└─────────┬────────────────┘
          ↓
┌─────────────────────────────────────────────────────────────────┐
│                Your OnSize handler executes                      │
│                                                                  │
│   Gui_Size(guiObj, MinMax, Width, Height) {                     │
│       if (MinMax = -1)  ; Minimized?                            │
│           return        ; Skip processing                       │
│                                                                  │
│       ; Calculate new positions                                 │
│       newEditWidth := Width - 20                                │
│       newEditHeight := Height - 60                              │
│       newBtnY := Height - 40                                    │
│                                                                  │
│       ; Move controls                                           │
│       EditBox.Move(, , newEditWidth, newEditHeight)             │
│       OKButton.Move(, newBtnY)                                  │
│   }                                                              │
└─────────┬───────────────────────────────────────────────────────┘
          ↓
┌─────────────────────────────────────────────────────────────────┐
│         Control.Move() calls SetWindowPos (Win32 API)           │
└─────────┬───────────────────────────────────────────────────────┘
          ↓
┌─────────────────────────────────────────────────────────────────┐
│              Windows redraws controls at new positions          │
│                   User sees updated layout                      │
└─────────────────────────────────────────────────────────────────┘
```

---

### Event Binding Resolution (Event Sink vs Bind)

```
══════════════════════════════════════════════════════════════════
                     EVENT SINK PATTERN
══════════════════════════════════════════════════════════════════

class MyDialog extends Gui {
    __New() {
        super.__New(opts, title, this) ← Event sink set to 'this'
        btn := this.Add("Button")
        btn.OnEvent("Click", "OnClick") ← String method name stored
    }

    OnClick(ctrl, info) { ... }
}

─────────────────────────────────────────────────────────────────

User clicks button
    ↓
Windows: WM_COMMAND message sent
    ↓
AHK2: Identify button control
    ↓
Retrieve OnEvent("Click") handler
    ↓
Handler is string: "OnClick"
    ↓
Lookup event sink object (stored during Gui.__New)
    ↓
Check: eventSink.HasMethod("OnClick")?
    ↓ YES
Call: eventSink.OnClick(ctrl, info)
    ↓
Method executes with 'this' = MyDialog instance


══════════════════════════════════════════════════════════════════
                        BIND() PATTERN
══════════════════════════════════════════════════════════════════

class MyDialog extends Gui {
    __New() {
        super.__New()  ← No event sink
        btn := this.Add("Button")
        fn := this.OnClick.Bind(this) ← Create bound function
        btn.OnEvent("Click", fn) ← Store function object
    }

    OnClick(ctrl, info) { ... }
}

─────────────────────────────────────────────────────────────────

User clicks button
    ↓
Windows: WM_COMMAND message sent
    ↓
AHK2: Identify button control
    ↓
Retrieve OnEvent("Click") handler
    ↓
Handler is function object (bound function)
    ↓
Call function: boundFn(ctrl, info)
    ↓
Bound function pre-fills 'this' parameter from Bind()
    ↓
Actual call: OnClick(prefilledThis, ctrl, info)
    ↓
Method executes with 'this' = value passed to Bind()


══════════════════════════════════════════════════════════════════
                         COMPARISON
══════════════════════════════════════════════════════════════════

Event Sink:
  Storage → String ("OnClick")
  Lookup  → Dynamic (eventSink.OnClick at call time)
  'this'  → Event sink object
  Polymorphism → YES (subclass can override OnClick)

Bind():
  Storage → BoundFunc object
  Lookup  → Direct (function already resolved)
  'this'  → Value passed to Bind()
  Polymorphism → NO (bound to specific method at bind time)
```

---

### MVC Message Flow

```
┌────────────────────────────────────────────────────────────────┐
│                         USER ACTION                             │
│              (clicks "Save" button in GUI)                      │
└─────────────────────────┬──────────────────────────────────────┘
                          ↓
╔═════════════════════════════════════════════════════════════════╗
║                          VIEW LAYER                              ║
╠═════════════════════════════════════════════════════════════════╣
║  class MyView {                                                 ║
║      OnSaveClick(ctrl, info) {                                  ║
║          inputValue := this.nameEdit.Value                      ║
║          this.controller.SaveData(inputValue) ← Delegate        ║
║      }                                                           ║
║  }                                                               ║
╚═════════════════════════┬═══════════════════════════════════════╝
                          ↓
╔═════════════════════════════════════════════════════════════════╗
║                      CONTROLLER LAYER                            ║
╠═════════════════════════════════════════════════════════════════╣
║  class Controller {                                             ║
║      SaveData(value) {                                          ║
║          ; Validate using Model                                 ║
║          if (!this.model.Validate(value)) {                     ║
║              this.view.ShowError("Invalid input") ← Update View ║
║              return                                             ║
║          }                                                       ║
║                                                                  ║
║          ; Process using Model                                  ║
║          processed := this.model.Process(value)                 ║
║                                                                  ║
║          ; Save using Model                                     ║
║          this.model.Save(processed)                             ║
║      }                                                           ║
║  }                                                               ║
╚════════════════════════┬════════════════════════════════════════╝
                         ↓
╔═════════════════════════════════════════════════════════════════╗
║                        MODEL LAYER                               ║
╠═════════════════════════════════════════════════════════════════╣
║  class DataModel {                                              ║
║      Validate(value) {                                          ║
║          return StrLen(value) > 0                               ║
║      }                                                           ║
║                                                                  ║
║      Process(value) {                                           ║
║          return StrUpper(value)                                 ║
║      }                                                           ║
║                                                                  ║
║      Save(value) {                                              ║
║          this.items.Push(value)                                 ║
║          this.NotifyObservers() ← Trigger update                ║
║      }                                                           ║
║                                                                  ║
║      NotifyObservers() {                                        ║
║          for observer in this.observers                         ║
║              observer(this.items) ← Callback                    ║
║      }                                                           ║
║  }                                                               ║
╚════════════════════════┬════════════════════════════════════════╝
                         ↓
            Observer callback fires
                         ↓
╔═════════════════════════════════════════════════════════════════╗
║                     CONTROLLER (Observer)                        ║
╠═════════════════════════════════════════════════════════════════╣
║  OnModelChanged(items) {                                        ║
║      this.view.UpdateListView(items) ← Update View              ║
║  }                                                               ║
╚════════════════════════┬════════════════════════════════════════╝
                         ↓
╔═════════════════════════════════════════════════════════════════╗
║                          VIEW LAYER                              ║
╠═════════════════════════════════════════════════════════════════╣
║  UpdateListView(items) {                                        ║
║      this.listView.Delete()  ; Clear                            ║
║      for item in items                                          ║
║          this.listView.Add("", item)  ; Populate                ║
║  }                                                               ║
╚═════════════════════════════════════════════════════════════════╝


DATA FLOW SUMMARY:

User Input → View → Controller → Model (validate/process/save)
                                    ↓
                            Model fires event
                                    ↓
                        Controller (observer) receives event
                                    ↓
                        Controller updates View
                                    ↓
                            User sees updated UI


DEPENDENCY DIAGRAM:

     ┌──────────────┐
     │  Controller  │ ← Knows both View and Model
     └───────┬──────┘
        ╱         ╲
       ↓           ↓
┌──────────┐  ┌──────────┐
│   View   │  │  Model   │ ← Neither knows the other
└──────────┘  └──────────┘

View depends on: Controller (calls its methods)
Model depends on: Nothing (just fires events)
Controller depends on: View & Model (orchestrates both)
```

---

### Component Composition Hierarchy

```
═══════════════════════════════════════════════════════════════════
                      COMPONENT TREE STRUCTURE
═══════════════════════════════════════════════════════════════════

SettingsDialog (Gui instance)
│
├─ Title (Text control)
│  └─ "Application Settings"
│
├─ PersonalSection (FormSection component) ───┐
│  │                                           │
│  ├─ SectionHeader (Text control)            │ Component
│  │  └─ "Personal Information"               │ encapsulates
│  │                                           │ multiple
│  ├─ Separator (Text control, line)          │ controls
│  │                                           │
│  ├─ NameInput (LabeledInput component) ───┐ │
│  │  │                                      │ │
│  │  ├─ Label (Text control)               │ │ Nested
│  │  │  └─ "Name:"                         │ │ component
│  │  │                                      │ │
│  │  ├─ Edit (Edit control)                │ │
│  │  │  └─ (user input field)              │ │
│  │  │                                      │ │
│  │  └─ ErrorLabel (Text control)          │ │
│  │     └─ (validation message, hidden)    │ │
│  │                                         ┘ │
│  ├─ EmailInput (LabeledInput component)     │
│  │  ├─ Label                                 │
│  │  ├─ Edit                                  │
│  │  └─ ErrorLabel                            │
│  │                                            │
│  └─ PhoneInput (LabeledInput component)      │
│     ├─ Label                                  │
│     ├─ Edit                                   │
│     └─ ErrorLabel                             │
│                                               ┘
├─ AccountSection (FormSection component)
│  ├─ SectionHeader → "Account Details"
│  ├─ Separator
│  ├─ UsernameInput (LabeledInput)
│  │  ├─ Label
│  │  ├─ Edit
│  │  └─ ErrorLabel
│  └─ PasswordInput (LabeledInput)
│     ├─ Label
│     ├─ Edit (Password type)
│     └─ ErrorLabel
│
├─ StatusBar (Text control)
│  └─ "Ready"
│
└─ ButtonPanel (ButtonPanel component)
   ├─ Separator (Text control, line)
   ├─ SaveButton (Button control)
   │  └─ "Save"
   ├─ ResetButton (Button control)
   │  └─ "Reset to Defaults"
   └─ CancelButton (Button control)
      └─ "Cancel"


═══════════════════════════════════════════════════════════════════
                     COMPONENT RESPONSIBILITIES
═══════════════════════════════════════════════════════════════════

┌────────────────────────────────────────────────────────────────┐
│ LabeledInput Component                                          │
├────────────────────────────────────────────────────────────────┤
│ Owns:     Label, Edit, ErrorLabel controls                     │
│ Exposes:  .Value property, .IsValid() method                   │
│ Hides:    Internal layout, error display logic                 │
│ Reusable: YES - works in any Gui                               │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ FormSection Component                                           │
├────────────────────────────────────────────────────────────────┤
│ Owns:     SectionHeader, Separator, child inputs               │
│ Exposes:  .AddField() method, .Enable()/.Disable()             │
│ Hides:    Separator styling, header formatting                 │
│ Reusable: YES - generic section container                      │
└────────────────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────────────────┐
│ ButtonPanel Component                                           │
├────────────────────────────────────────────────────────────────┤
│ Owns:     Separator, Button controls                           │
│ Exposes:  .AddButton(text, callback) method                    │
│ Hides:    Button positioning, styling consistency              │
│ Reusable: YES - standard button bar for dialogs                │
└────────────────────────────────────────────────────────────────┘


═══════════════════════════════════════════════════════════════════
              COMPOSITION vs INHERITANCE COMPARISON
═══════════════════════════════════════════════════════════════════

COMPOSITION (used above):
  Structure → Component HAS-A control
  Example   → LabeledInput has-a Edit control
  Lifetime  → Component manages control lifecycle
  Embedding → Components added to parent Gui
  Reuse     → Same component in multiple GUIs

INHERITANCE (alternative):
  Structure → Component IS-A Gui
  Example   → LabeledInput extends Gui
  Lifetime  → Each component is separate window
  Embedding → Cannot embed (each is top-level)
  Reuse     → Only as standalone window

Composition wins for components because:
  ✓ Multiple components in single window
  ✓ Logical grouping without window overhead
  ✓ Parent Gui controls overall layout
  ✓ Component focuses on behavior, not windowing
```

---

## Conclusion: The Architecture Philosophy

AHK2's GUI system is built on **pragmatic minimalism**:

1. **Object-oriented foundation** enables composition and reuse
2. **Manual layout system** prioritizes transparency over convenience
3. **Event sink pattern** provides clean class-based event handling
4. **Method chaining** makes builder APIs readable
5. **Component composition** structures complex interfaces
6. **MVC separation** keeps business logic testable
7. **Observable properties** simplify state synchronization
8. **Community libraries** fill framework gaps without bloating core

The design philosophy: **Provide primitives, not frameworks.** Give developers full control through simple, transparent mechanisms. Let the community build sophisticated abstractions on top.

This architecture rewards **understanding over memorization**. Once you grasp the underlying patterns—object lifecycles, event dispatch, property accessors, composition—you can build GUI systems as simple or sophisticated as your needs require.

The "gaps" (no layout managers, no data binding, no theming engine) are **intentional**. They keep the core maintainable, the runtime lean, and the design space open for experimentation. The thriving library ecosystem proves this approach works: when the foundation is solid, the community builds cathedrals.
