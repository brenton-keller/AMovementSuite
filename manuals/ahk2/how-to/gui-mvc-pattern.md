# How-To: Implement MVC Pattern for GUI Applications

## Problem

Your GUI code is becoming a tangled mess. Business logic is mixed with button handlers, validation is scattered across event callbacks, and changing one feature breaks three others. Testing is nearly impossible because everything depends on the GUI. The codebase has become unmaintainable as it grows.

## Quick Context

The **Model-View-Controller (MVC)** pattern separates concerns into three distinct layers:

- **Model**: Business logic, data storage, validation rules (knows nothing about GUI)
- **View**: GUI presentation, user input, display updates (knows nothing about business rules)
- **Controller**: Coordinates between Model and View, handles user actions

**Why separate?** Clean separation enables:
- Testing business logic without GUI
- Changing UI without touching logic
- Multiple views for same data
- Clear responsibilities and maintainability

In AHK2, this translates to using classes for each layer with clear interfaces between them.

For deeper architectural understanding, see:
- docs/manuals/ahk2/explanation/gui-architecture.md
- docs/manuals/ahk2/explanation/oop-patterns.md

## Key Concepts

- **Model**: Pure data and logic, no GUI dependencies, observable for change notifications
- **View**: Extends `Gui` class, only handles display and user input collection
- **Controller**: Instantiates Model and View, connects them, handles coordination
- **Observer Pattern**: Model notifies View of data changes automatically
- **Dependency Injection**: View receives Controller reference, Controller owns Model

**Data flow**:
```
User Input → View → Controller → Model → Observer Notification → View Update
```

## Solution 1: Basic MVC Structure

**Use when**: Starting a new GUI application with moderate complexity.

```ahk
#Requires AutoHotkey v2.0

; =============================================================================
; MODEL - Business Logic Only
; =============================================================================

class TaskModel {
    __New() {
        this.tasks := []
        this.observers := []
    }

    AddTask(title, priority) {
        ; Validation in model
        if (StrLen(Trim(title)) = 0)
            throw Error("Task title cannot be empty")

        if (priority < 1 || priority > 5)
            throw Error("Priority must be between 1 and 5")

        ; Business logic
        task := {
            id: this.GenerateID(),
            title: title,
            priority: priority,
            completed: false,
            created: A_Now
        }

        this.tasks.Push(task)
        this.NotifyObservers()
        return task
    }

    MarkComplete(taskID) {
        for task in this.tasks {
            if (task.id = taskID) {
                task.completed := true
                this.NotifyObservers()
                return true
            }
        }
        return false
    }

    DeleteTask(taskID) {
        for i, task in this.tasks {
            if (task.id = taskID) {
                this.tasks.RemoveAt(i)
                this.NotifyObservers()
                return true
            }
        }
        return false
    }

    GetAllTasks() {
        return this.tasks.Clone()
    }

    GetPendingTasks() {
        pending := []
        for task in this.tasks
            if (!task.completed)
                pending.Push(task)
        return pending
    }

    ; Observer pattern for View updates
    Subscribe(observer) {
        this.observers.Push(observer)
    }

    NotifyObservers() {
        for observer in this.observers
            observer.Call()
    }

    ; Helper
    GenerateID() {
        static counter := 0
        return ++counter
    }
}

; =============================================================================
; VIEW - GUI Presentation Only
; =============================================================================

class TaskView extends Gui {
    __New(controller) {
        super.__New("+Resize", "Task Manager", this)

        this.controller := controller
        this.CreateControls()
        this.OnEvent("Close", (*) => ExitApp)
        this.Show("w500 h400")
    }

    CreateControls() {
        this.SetFont("s10", "Segoe UI")

        ; Input section
        this.Add("Text", "xm", "Task Title:")
        this.titleEdit := this.Add("Edit", "x+10 w300")

        this.Add("Text", "xm", "Priority (1-5):")
        this.priorityEdit := this.Add("Edit", "x+10 w60 Number")
        this.priorityEdit.Value := 3

        this.addBtn := this.Add("Button", "xm w150 Default", "Add Task")
        this.addBtn.OnEvent("Click", (*) => this.OnAddTask())

        ; Task list
        this.Add("Text", "xm w480 h2 +0x10")
        this.taskList := this.Add("ListView", "xm w480 h250",
            ["ID", "Task", "Priority", "Status"])
        this.taskList.ModifyCol(1, 50)
        this.taskList.ModifyCol(2, 250)
        this.taskList.ModifyCol(3, 80)
        this.taskList.ModifyCol(4, 100)

        ; Action buttons
        this.completeBtn := this.Add("Button", "xm w150", "Mark Complete")
        this.deleteBtn := this.Add("Button", "x+10 w150", "Delete Task")

        this.completeBtn.OnEvent("Click", (*) => this.OnMarkComplete())
        this.deleteBtn.OnEvent("Click", (*) => this.OnDeleteTask())
    }

    ; View delegates user actions to Controller
    OnAddTask() {
        this.controller.AddTask(this.titleEdit.Value, this.priorityEdit.Value)
    }

    OnMarkComplete() {
        row := this.taskList.GetNext()
        if (row)
            this.controller.MarkComplete(this.taskList.GetText(row, 1))
    }

    OnDeleteTask() {
        row := this.taskList.GetNext()
        if (row)
            this.controller.DeleteTask(this.taskList.GetText(row, 1))
    }

    ; View only handles display updates
    UpdateTaskList(tasks) {
        this.taskList.Delete()

        for task in tasks {
            status := task.completed ? "Complete" : "Pending"
            this.taskList.Add("", task.id, task.title, task.priority, status)
        }
    }

    ClearInput() {
        this.titleEdit.Value := ""
        this.priorityEdit.Value := 3
        this.titleEdit.Focus()
    }

    ShowError(message) {
        MsgBox(message, "Error", 16)
    }

    ShowSuccess(message) {
        MsgBox(message, "Success", 64)
    }
}

; =============================================================================
; CONTROLLER - Coordination
; =============================================================================

class TaskController {
    __New() {
        ; Create Model
        this.model := TaskModel()

        ; Create View
        this.view := TaskView(this)

        ; Subscribe View to Model changes
        this.model.Subscribe(() => this.UpdateView())

        ; Initial display
        this.UpdateView()
    }

    AddTask(title, priority) {
        try {
            task := this.model.AddTask(title, priority)
            this.view.ClearInput()
            this.view.ShowSuccess("Task added: " task.title)
        } catch Error as e {
            this.view.ShowError(e.Message)
        }
    }

    MarkComplete(taskID) {
        if (this.model.MarkComplete(taskID))
            this.view.ShowSuccess("Task marked complete!")
        else
            this.view.ShowError("Task not found")
    }

    DeleteTask(taskID) {
        result := MsgBox("Delete this task?", "Confirm", "YesNo Icon?")
        if (result = "Yes") {
            if (this.model.DeleteTask(taskID))
                this.view.ShowSuccess("Task deleted")
            else
                this.view.ShowError("Task not found")
        }
    }

    UpdateView() {
        tasks := this.model.GetAllTasks()
        this.view.UpdateTaskList(tasks)
    }
}

; =============================================================================
; APPLICATION ENTRY POINT
; =============================================================================

app := TaskController()
```

**What this demonstrates**:
- **Model** has zero GUI dependencies—purely business logic
- **View** only collects input and displays data
- **Controller** coordinates between them
- **Observer pattern** automatically updates View when Model changes
- Clean separation enables testing Model independently

## Solution 2: Advanced MVC with Data Binding

**Use when**: Complex data flows, multiple views, or real-time updates needed.

```ahk
#Requires AutoHotkey v2.0

; =============================================================================
; OBSERVABLE MODEL - Automatic View Updates
; =============================================================================

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
                this.Notify()
            }
        }
    }

    Subscribe(observer) {
        this._observers.Push(observer)
        return this
    }

    Notify() {
        for observer in this._observers
            observer(this._value)
    }
}

class UserProfileModel {
    __New() {
        ; Observable properties
        this.username := ObservableProperty("Guest")
        this.email := ObservableProperty("")
        this.premium := ObservableProperty(false)
        this.loginCount := ObservableProperty(0)

        ; Computed properties
        this.displayName := ObservableProperty("Guest User")

        ; Link computed property to dependencies
        this.username.Subscribe((val) => this.UpdateDisplayName())
        this.premium.Subscribe((val) => this.UpdateDisplayName())
    }

    UpdateDisplayName() {
        prefix := this.premium.Value ? "⭐ " : ""
        this.displayName.Value := prefix this.username.Value
    }

    Validate() {
        errors := []

        if (StrLen(Trim(this.username.Value)) < 3)
            errors.Push("Username must be at least 3 characters")

        if (!InStr(this.email.Value, "@") || !InStr(this.email.Value, "."))
            errors.Push("Invalid email address")

        return errors
    }

    Save() {
        ; Simulate API call
        Sleep(500)
        this.loginCount.Value++
        return true
    }
}

; =============================================================================
; VIEW WITH DATA BINDING
; =============================================================================

class ProfileView extends Gui {
    __New(controller) {
        super.__New("+Resize", "User Profile", this)

        this.controller := controller
        this.CreateControls()
        this.BindData()
        this.OnEvent("Close", (*) => ExitApp)
        this.Show("w500 h400")
    }

    CreateControls() {
        this.SetFont("s10", "Segoe UI")

        ; Display name (read-only, auto-updates)
        this.SetFont("s14 Bold")
        this.displayLabel := this.Add("Text", "xm w480 Center", "")
        this.SetFont("s10")

        ; Input fields
        this.Add("Text", "xm w480 h2 +0x10")
        this.Add("Text", "xm", "Username:")
        this.usernameEdit := this.Add("Edit", "x+10 w300")

        this.Add("Text", "xm", "Email:")
        this.emailEdit := this.Add("Edit", "x+10 w300")

        this.premiumCheck := this.Add("Checkbox", "xm", "Premium Account")

        ; Stats (read-only, auto-updates)
        this.Add("Text", "xm w480 h2 +0x10")
        this.statsLabel := this.Add("Text", "xm w480", "")

        ; Error display
        this.errorLabel := this.Add("Text", "xm w480 h60 cRed", "")

        ; Buttons
        this.saveBtn := this.Add("Button", "xm w150 Default", "Save Profile")
        this.resetBtn := this.Add("Button", "x+10 w150", "Reset")

        this.saveBtn.OnEvent("Click", (*) => this.OnSave())
        this.resetBtn.OnEvent("Click", (*) => this.OnReset())

        ; Watch for input changes
        this.usernameEdit.OnEvent("Change", (*) => this.OnFieldChange("username"))
        this.emailEdit.OnEvent("Change", (*) => this.OnFieldChange("email"))
        this.premiumCheck.OnEvent("Click", (*) => this.OnFieldChange("premium"))
    }

    BindData() {
        model := this.controller.model

        ; Bind model properties to view updates
        model.displayName.Subscribe((val) => this.displayLabel.Text := val)
        model.loginCount.Subscribe((val) => this.UpdateStats())

        ; Initial sync
        this.usernameEdit.Value := model.username.Value
        this.emailEdit.Value := model.email.Value
        this.premiumCheck.Value := model.premium.Value
        this.UpdateStats()
    }

    OnFieldChange(field) {
        ; Sync view changes to model
        model := this.controller.model

        switch field {
            case "username": model.username.Value := this.usernameEdit.Value
            case "email": model.email.Value := this.emailEdit.Value
            case "premium": model.premium.Value := this.premiumCheck.Value
        }
    }

    OnSave() {
        this.controller.SaveProfile()
    }

    OnReset() {
        this.controller.ResetProfile()
    }

    UpdateStats() {
        count := this.controller.model.loginCount.Value
        this.statsLabel.Text := "Login count: " count
    }

    ShowErrors(errors) {
        this.errorLabel.Text := errors.Length > 0 ? errors.Join("`n") : ""
    }

    ClearErrors() {
        this.errorLabel.Text := ""
    }

    ShowSaving() {
        this.saveBtn.Enabled := false
        this.saveBtn.Text := "Saving..."
    }

    ShowSaved() {
        this.saveBtn.Enabled := true
        this.saveBtn.Text := "Save Profile"
        MsgBox("Profile saved successfully!", "Success", 64)
    }
}

; =============================================================================
; CONTROLLER WITH DATA BINDING
; =============================================================================

class ProfileController {
    __New() {
        this.model := UserProfileModel()
        this.view := ProfileView(this)
    }

    SaveProfile() {
        ; Validate through model
        errors := this.model.Validate()

        if (errors.Length > 0) {
            this.view.ShowErrors(errors)
            return
        }

        this.view.ClearErrors()
        this.view.ShowSaving()

        ; Save (async simulation)
        SetTimer(() => this.CompleteSave(), -500)
    }

    CompleteSave() {
        success := this.model.Save()
        if (success)
            this.view.ShowSaved()
    }

    ResetProfile() {
        this.model.username.Value := "Guest"
        this.model.email.Value := ""
        this.model.premium.Value := false

        ; View auto-updates through bindings
        this.view.usernameEdit.Value := this.model.username.Value
        this.view.emailEdit.Value := this.model.email.Value
        this.view.premiumCheck.Value := this.model.premium.Value
        this.view.ClearErrors()
    }
}

; Launch
app := ProfileController()
```

**Advanced features demonstrated**:
- **Observable properties** trigger automatic View updates
- **Computed properties** (displayName) auto-update from dependencies
- **Two-way binding** between View inputs and Model properties
- **Reactive updates** without manual refresh calls
- **Clean validation** with automatic error display

## Solution 3: MVC with Persistence Layer

**Use when**: Data needs to be saved/loaded from files, databases, or APIs.

```ahk
#Requires AutoHotkey v2.0

; =============================================================================
; MODEL with Business Logic
; =============================================================================

class NoteModel {
    __New() {
        this.notes := []
        this.observers := []
        this.currentNote := ""
        this.isDirty := false
    }

    CreateNote(title, content) {
        note := {
            id: this.GenerateID(),
            title: title,
            content: content,
            created: A_Now,
            modified: A_Now
        }

        this.notes.Push(note)
        this.NotifyObservers()
        return note
    }

    UpdateNote(id, title, content) {
        for note in this.notes {
            if (note.id = id) {
                note.title := title
                note.content := content
                note.modified := A_Now
                this.NotifyObservers()
                return true
            }
        }
        return false
    }

    DeleteNote(id) {
        for i, note in this.notes {
            if (note.id = id) {
                this.notes.RemoveAt(i)
                this.NotifyObservers()
                return true
            }
        }
        return false
    }

    GetNote(id) {
        for note in this.notes
            if (note.id = id)
                return note
        return ""
    }

    GetAllNotes() {
        return this.notes.Clone()
    }

    Subscribe(observer) {
        this.observers.Push(observer)
    }

    NotifyObservers() {
        for observer in this.observers
            observer.Call()
    }

    GenerateID() {
        static counter := 0
        return ++counter
    }
}

; =============================================================================
; PERSISTENCE LAYER - Data Storage
; =============================================================================

class NoteRepository {
    __New(filename := "notes.json") {
        this.filename := filename
    }

    Save(notes) {
        ; Convert to JSON and save
        json := this.NotesToJSON(notes)

        try {
            file := FileOpen(this.filename, "w")
            file.Write(json)
            file.Close()
            return true
        } catch {
            return false
        }
    }

    Load() {
        if (!FileExist(this.filename))
            return []

        try {
            file := FileOpen(this.filename, "r")
            json := file.Read()
            file.Close()
            return this.JSONToNotes(json)
        } catch {
            return []
        }
    }

    NotesToJSON(notes) {
        ; Simple JSON serialization
        json := "["

        for i, note in notes {
            if (i > 1)
                json .= ","

            json .= '{'
            json .= '"id":' note.id ','
            json .= '"title":"' this.EscapeJSON(note.title) '",'
            json .= '"content":"' this.EscapeJSON(note.content) '",'
            json .= '"created":"' note.created '",'
            json .= '"modified":"' note.modified '"'
            json .= '}'
        }

        json .= "]"
        return json
    }

    JSONToNotes(json) {
        ; Simple JSON parsing (production would use proper JSON library)
        notes := []

        ; This is simplified - use a proper JSON parser in production
        ; For demonstration purposes only
        Loop Parse json, "{}" {
            if (A_LoopField = "" || A_LoopField = "[" || A_LoopField = "]" || A_LoopField = ",")
                continue

            ; Extract fields (simplified)
            note := {}
            ; In production: use proper JSON parsing library
        }

        return notes
    }

    EscapeJSON(str) {
        str := StrReplace(str, "`"", "\`"")
        str := StrReplace(str, "`n", "\n")
        str := StrReplace(str, "`r", "\r")
        return str
    }
}

; =============================================================================
; VIEW - GUI Only
; =============================================================================

class NoteView extends Gui {
    __New(controller) {
        super.__New("+Resize", "Note Manager", this)

        this.controller := controller
        this.CreateControls()
        this.OnEvent("Close", (*) => this.OnClose())
        this.OnEvent("Size", this.HandleResize.Bind(this))
        this.Show("w700 h500")
    }

    CreateControls() {
        ; Note list
        this.noteList := this.Add("ListView", "x10 y10 w200 h440", ["Notes"])
        this.noteList.OnEvent("Click", (*) => this.OnNoteSelect())

        ; Note editor
        this.Add("Text", "x220 y10", "Title:")
        this.titleEdit := this.Add("Edit", "x+10 w450")

        this.Add("Text", "x220 y+10", "Content:")
        this.contentEdit := this.Add("Edit", "x220 y+5 w460 h370 +Multi")

        ; Buttons
        this.newBtn := this.Add("Button", "x10 y+10 w90", "New")
        this.saveBtn := this.Add("Button", "x+10 w90", "Save")
        this.deleteBtn := this.Add("Button", "x+10 w90", "Delete")

        this.newBtn.OnEvent("Click", (*) => this.OnNew())
        this.saveBtn.OnEvent("Click", (*) => this.OnSave())
        this.deleteBtn.OnEvent("Click", (*) => this.OnDelete())

        ; Track changes
        this.titleEdit.OnEvent("Change", (*) => this.OnContentChange())
        this.contentEdit.OnEvent("Change", (*) => this.OnContentChange())
    }

    OnNoteSelect() {
        row := this.noteList.GetNext()
        if (row)
            this.controller.SelectNote(this.noteList.GetText(row, 1))
    }

    OnNew() {
        this.controller.CreateNewNote()
    }

    OnSave() {
        this.controller.SaveCurrentNote(this.titleEdit.Value, this.contentEdit.Value)
    }

    OnDelete() {
        row := this.noteList.GetNext()
        if (row)
            this.controller.DeleteNote(this.noteList.GetText(row, 1))
    }

    OnContentChange() {
        this.controller.MarkDirty()
    }

    OnClose() {
        if (this.controller.HasUnsavedChanges()) {
            result := MsgBox("Save changes before closing?", "Unsaved Changes", "YesNoCancel")
            if (result = "Cancel")
                return true  ; Prevent close
            if (result = "Yes")
                this.controller.SaveAll()
        }
        ExitApp()
    }

    UpdateNoteList(notes) {
        this.noteList.Delete()
        for note in notes
            this.noteList.Add("", note.title ? note.title : "(Untitled)")
    }

    DisplayNote(note) {
        this.titleEdit.Value := note ? note.title : ""
        this.contentEdit.Value := note ? note.content : ""
    }

    ClearEditor() {
        this.titleEdit.Value := ""
        this.contentEdit.Value := ""
    }

    HandleResize(guiObj, MinMax, Width, Height) {
        if (MinMax = -1)
            return

        this.noteList.Move(, , , Height - 70)
        this.contentEdit.Move(, , Width - 230, Height - 140)
        this.titleEdit.Move(, , Width - 230)
    }

    ShowMessage(msg, title := "Note Manager", icon := 64) {
        MsgBox(msg, title, icon)
    }
}

; =============================================================================
; CONTROLLER - Coordination + Persistence
; =============================================================================

class NoteController {
    __New() {
        this.model := NoteModel()
        this.repo := NoteRepository("notes.json")
        this.view := NoteView(this)
        this.currentNoteID := ""
        this.isDirty := false

        ; Load existing notes
        this.LoadNotes()

        ; Subscribe to model changes
        this.model.Subscribe(() => this.OnModelChange())
    }

    LoadNotes() {
        notes := this.repo.Load()
        this.model.notes := notes
        this.view.UpdateNoteList(notes)
    }

    CreateNewNote() {
        if (this.isDirty) {
            result := MsgBox("Save current note?", "Unsaved Changes", "YesNoCancel")
            if (result = "Cancel")
                return
            if (result = "Yes")
                this.SaveCurrentNote(this.view.titleEdit.Value, this.view.contentEdit.Value)
        }

        this.currentNoteID := ""
        this.view.ClearEditor()
        this.isDirty := false
    }

    SelectNote(title) {
        ; Find note by title
        for note in this.model.GetAllNotes() {
            if (note.title = title || (!note.title && title = "(Untitled)")) {
                this.currentNoteID := note.id
                this.view.DisplayNote(note)
                this.isDirty := false
                return
            }
        }
    }

    SaveCurrentNote(title, content) {
        if (this.currentNoteID) {
            ; Update existing
            this.model.UpdateNote(this.currentNoteID, title, content)
        } else {
            ; Create new
            note := this.model.CreateNote(title, content)
            this.currentNoteID := note.id
        }

        this.SaveAll()
        this.isDirty := false
        this.view.ShowMessage("Note saved!")
    }

    DeleteNote(title) {
        result := MsgBox("Delete this note?", "Confirm", "YesNo Icon?")
        if (result = "No")
            return

        for note in this.model.GetAllNotes() {
            if (note.title = title || (!note.title && title = "(Untitled)")) {
                this.model.DeleteNote(note.id)
                this.SaveAll()
                this.view.ClearEditor()
                this.currentNoteID := ""
                this.isDirty := false
                return
            }
        }
    }

    SaveAll() {
        success := this.repo.Save(this.model.GetAllNotes())
        if (!success)
            this.view.ShowMessage("Failed to save notes!", "Error", 16)
    }

    MarkDirty() {
        this.isDirty := true
    }

    HasUnsavedChanges() {
        return this.isDirty
    }

    OnModelChange() {
        this.view.UpdateNoteList(this.model.GetAllNotes())
    }
}

; Launch
app := NoteController()
```

**What this adds**:
- **Repository pattern** separates data storage from business logic
- **Persistence** with file-based storage (easily swapped for DB/API)
- **Dirty tracking** to prevent data loss
- **Load/Save lifecycle** managed by Controller
- **Clean architecture** with clear layer boundaries

## Complete Working Example: Contact Manager with Full MVC

```ahk
#Requires AutoHotkey v2.0
#SingleInstance Force

; =============================================================================
; CONTACT MANAGER - Complete MVC Application
; Demonstrates: Full MVC pattern with validation, persistence, observer pattern
; =============================================================================

; MODEL - Business Logic
class Contact {
    __New(name, email, phone) {
        this.id := Random(100000, 999999)
        this.name := name
        this.email := email
        this.phone := phone
        this.created := A_Now
    }
}

class ContactModel {
    __New() {
        this.contacts := []
        this.observers := []
    }

    Add(name, email, phone) {
        errors := this.Validate(name, email, phone)
        if (errors.Length > 0)
            throw Error(errors.Join("`n"))

        contact := Contact(name, email, phone)
        this.contacts.Push(contact)
        this.NotifyObservers()
        return contact
    }

    Update(id, name, email, phone) {
        errors := this.Validate(name, email, phone)
        if (errors.Length > 0)
            throw Error(errors.Join("`n"))

        for contact in this.contacts {
            if (contact.id = id) {
                contact.name := name
                contact.email := email
                contact.phone := phone
                this.NotifyObservers()
                return true
            }
        }
        return false
    }

    Delete(id) {
        for i, contact in this.contacts {
            if (contact.id = id) {
                this.contacts.RemoveAt(i)
                this.NotifyObservers()
                return true
            }
        }
        return false
    }

    Get(id) {
        for contact in this.contacts
            if (contact.id = id)
                return contact
        return ""
    }

    GetAll() => this.contacts.Clone()

    Search(query) {
        results := []
        query := StrLower(query)

        for contact in this.contacts {
            if (InStr(StrLower(contact.name), query)
                || InStr(StrLower(contact.email), query)
                || InStr(StrLower(contact.phone), query))
                results.Push(contact)
        }

        return results
    }

    Validate(name, email, phone) {
        errors := []

        if (StrLen(Trim(name)) = 0)
            errors.Push("Name is required")

        if (!InStr(email, "@") || !InStr(email, "."))
            errors.Push("Invalid email address")

        phoneDigits := RegExReplace(phone, "\D")
        if (StrLen(phoneDigits) < 10)
            errors.Push("Phone must have at least 10 digits")

        return errors
    }

    Subscribe(observer) {
        this.observers.Push(observer)
    }

    NotifyObservers() {
        for observer in this.observers
            observer.Call()
    }
}

; VIEW - GUI Only
class ContactView extends Gui {
    __New(controller) {
        super.__New("+Resize +MinSize600x400", "Contact Manager", this)

        this.controller := controller
        this.CreateControls()
        this.OnEvent("Close", (*) => ExitApp)
        this.OnEvent("Size", this.HandleResize.Bind(this))
        this.Show("w700 h500")
    }

    CreateControls() {
        ; Search
        this.Add("Text", "xm", "Search:")
        this.searchEdit := this.Add("Edit", "x+10 w300")
        this.searchBtn := this.Add("Button", "x+10 w100", "Search")
        this.searchBtn.OnEvent("Click", (*) => this.OnSearch())

        ; Contact list
        this.contactList := this.Add("ListView", "xm w680 h250",
            ["ID", "Name", "Email", "Phone"])
        this.contactList.ModifyCol(1, 0)  ; Hide ID
        this.contactList.ModifyCol(2, 200)
        this.contactList.ModifyCol(3, 250)
        this.contactList.ModifyCol(4, 200)
        this.contactList.OnEvent("Click", (*) => this.OnSelectContact())

        ; Editor
        this.Add("Text", "xm w680 h2 +0x10")
        this.Add("Text", "xm", "Name:")
        this.nameEdit := this.Add("Edit", "x+10 w250")

        this.Add("Text", "xm", "Email:")
        this.emailEdit := this.Add("Edit", "x+10 w250")

        this.Add("Text", "xm", "Phone:")
        this.phoneEdit := this.Add("Edit", "x+10 w250")

        ; Buttons
        this.newBtn := this.Add("Button", "xm w100", "New")
        this.saveBtn := this.Add("Button", "x+10 w100 Disabled", "Save")
        this.deleteBtn := this.Add("Button", "x+10 w100 Disabled", "Delete")
        this.clearBtn := this.Add("Button", "x+10 w100", "Clear")

        this.newBtn.OnEvent("Click", (*) => this.OnNew())
        this.saveBtn.OnEvent("Click", (*) => this.OnSave())
        this.deleteBtn.OnEvent("Click", (*) => this.OnDelete())
        this.clearBtn.OnEvent("Click", (*) => this.OnClear())
    }

    OnSearch() {
        this.controller.Search(this.searchEdit.Value)
    }

    OnSelectContact() {
        row := this.contactList.GetNext()
        if (row) {
            id := this.contactList.GetText(row, 1)
            this.controller.SelectContact(id)
        }
    }

    OnNew() {
        this.controller.CreateNew()
    }

    OnSave() {
        this.controller.Save(
            this.nameEdit.Value,
            this.emailEdit.Value,
            this.phoneEdit.Value
        )
    }

    OnDelete() {
        row := this.contactList.GetNext()
        if (row) {
            id := this.contactList.GetText(row, 1)
            this.controller.Delete(id)
        }
    }

    OnClear() {
        this.ClearForm()
    }

    UpdateContactList(contacts) {
        this.contactList.Delete()
        for contact in contacts
            this.contactList.Add("", contact.id, contact.name, contact.email, contact.phone)
    }

    DisplayContact(contact) {
        this.nameEdit.Value := contact.name
        this.emailEdit.Value := contact.email
        this.phoneEdit.Value := contact.phone
        this.saveBtn.Enabled := true
        this.deleteBtn.Enabled := true
    }

    ClearForm() {
        this.nameEdit.Value := ""
        this.emailEdit.Value := ""
        this.phoneEdit.Value := ""
        this.saveBtn.Enabled := false
        this.deleteBtn.Enabled := false
        this.searchEdit.Value := ""
    }

    EnableNewMode() {
        this.ClearForm()
        this.saveBtn.Enabled := true
        this.nameEdit.Focus()
    }

    ShowError(msg) {
        MsgBox(msg, "Error", 16)
    }

    ShowSuccess(msg) {
        MsgBox(msg, "Success", 64)
    }

    HandleResize(guiObj, MinMax, Width, Height) {
        if (MinMax = -1)
            return

        this.contactList.Move(, , Width - 20, Height - 200)
    }
}

; CONTROLLER - Coordination
class ContactController {
    __New() {
        this.model := ContactModel()
        this.view := ContactView(this)
        this.currentContactID := ""

        this.model.Subscribe(() => this.UpdateView())
        this.UpdateView()
    }

    CreateNew() {
        this.currentContactID := ""
        this.view.EnableNewMode()
    }

    SelectContact(id) {
        contact := this.model.Get(id)
        if (contact) {
            this.currentContactID := id
            this.view.DisplayContact(contact)
        }
    }

    Save(name, email, phone) {
        try {
            if (this.currentContactID) {
                this.model.Update(this.currentContactID, name, email, phone)
                this.view.ShowSuccess("Contact updated!")
            } else {
                contact := this.model.Add(name, email, phone)
                this.currentContactID := contact.id
                this.view.ShowSuccess("Contact created!")
            }
            this.view.ClearForm()
            this.currentContactID := ""
        } catch Error as e {
            this.view.ShowError(e.Message)
        }
    }

    Delete(id) {
        result := MsgBox("Delete this contact?", "Confirm", "YesNo Icon?")
        if (result = "Yes") {
            this.model.Delete(id)
            this.view.ClearForm()
            this.currentContactID := ""
            this.view.ShowSuccess("Contact deleted")
        }
    }

    Search(query) {
        if (query = "") {
            this.UpdateView()
        } else {
            results := this.model.Search(query)
            this.view.UpdateContactList(results)
        }
    }

    UpdateView() {
        this.view.UpdateContactList(this.model.GetAll())
    }
}

; Launch
app := ContactController()

; Development hotkey
^R:: Reload
```

## Troubleshooting

### Problem: Model has GUI dependencies
**Solution**: If you're using `MsgBox` or GUI controls in Model, you're doing it wrong. Move all UI interactions to View or Controller.

### Problem: View contains business logic
**Solution**: Validation, calculations, data transformations belong in Model. View should only format for display.

### Problem: Controller becoming a "god object"
**Solution**: If Controller is too large, split into multiple Controllers or introduce a Service layer between Controller and Model.

### Problem: Circular references between layers
**Solution**: Only Controller should reference both Model and View. Model and View should not reference each other directly.

### Problem: Testing is difficult
**Solution**: Model should be testable without any GUI. If you can't instantiate and test Model alone, refactor dependencies out.

## Best Practices

1. **Model has zero GUI code**: No `MsgBox`, no control references
2. **View delegates logic**: All decisions go through Controller to Model
3. **Controller is thin**: Mostly coordination, minimal logic
4. **Observer pattern for updates**: Model notifies Views automatically
5. **Validation in Model**: Business rules belong in Model
6. **Single Responsibility**: Each class has one clear purpose
7. **Dependency Injection**: Pass dependencies via constructor
8. **Interface segregation**: Models expose only what Views need
9. **Test Models independently**: Write unit tests without GUI
10. **Document data flow**: Comment how data moves between layers

## When to Use MVC

| Scenario | Use MVC? | Why |
|----------|----------|-----|
| **Simple utility (<100 lines)** | No | Overhead outweighs benefits |
| **Form-based app (200-500 lines)** | Maybe | Consider simpler separation |
| **Multi-window application** | Yes | Shared Model, multiple Views |
| **Complex business logic** | Yes | Testability is critical |
| **Team development** | Yes | Clear contracts between layers |
| **Long-term maintenance** | Yes | Easier to modify over time |

## Further Reading

- docs/manuals/ahk2/explanation/oop-patterns.md - OOP design patterns
- docs/manuals/ahk2/explanation/gui-architecture.md - GUI architecture concepts
- docs/manuals/ahk2/how-to/reusable-gui-components.md - Component patterns
- docs/manuals/ahk2/reference/gui-class.md - Gui class reference

## Related Community Resources

- **AHK v2 GUI Megathread**: https://www.autohotkey.com/boards/viewtopic.php?t=123220
- **Design Patterns Discussion**: https://www.autohotkey.com/boards/viewtopic.php?f=83&t=94669
