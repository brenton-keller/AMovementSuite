# Tutorial: Intermediate AutoHotkey v2 Concepts

Now that you understand the basics, let's explore intermediate concepts: classes, objects, closures, and error handling.

## Prerequisites

- Completion of docs/manuals/ahk2/tutorials/01-getting-started.md
- Understanding of functions and variables
- Familiarity with basic programming concepts

## Understanding Objects

Objects store related data together. Think of them as containers with labeled compartments:

```ahk
#Requires AutoHotkey v2.0

; Create a simple object
person := {
    name: "Alice",
    age: 30,
    city: "Seattle"
}

; Access properties
MsgBox person.name  ; Shows "Alice"
MsgBox person.age   ; Shows 30

; Add new properties
person.job := "Engineer"

; Modify existing properties
person.age := 31
```

**Object syntax:**
- `{}` creates an object
- `property: value` defines properties
- `.` accesses properties
- Properties can be added anytime

### Arrays: Ordered Collections

Arrays store multiple items in order:

```ahk
#Requires AutoHotkey v2.0

; Create an array
colors := ["Red", "Green", "Blue"]

; Access by index (starts at 1, not 0!)
MsgBox colors[1]  ; Shows "Red"
MsgBox colors[2]  ; Shows "Green"

; Add items
colors.Push("Yellow")

; Get array length
MsgBox "Array has " colors.Length " items"

; Loop through array
for index, color in colors {
    MsgBox "Color " index ": " color
}
```

**Key points:**
- Arrays use `[]` for creation and access
- Indices start at 1 (unlike many languages)
- `.Push()` adds to end
- `.Length` gets count

### Maps: Key-Value Pairs

Maps associate keys with values (like a dictionary):

```ahk
#Requires AutoHotkey v2.0

; Create a map
phoneBook := Map(
    "Alice", "555-1234",
    "Bob", "555-5678",
    "Charlie", "555-9012"
)

; Access by key
MsgBox phoneBook["Alice"]  ; Shows "555-1234"

; Add or update entries
phoneBook["David"] := "555-3456"

; Check if key exists
if phoneBook.Has("Alice")
    MsgBox "Alice is in the phone book"

; Loop through map
for name, number in phoneBook {
    MsgBox name ": " number
}
```

**Map features:**
- Keys can be strings or numbers
- `.Has(key)` checks existence
- `[key]` accesses values
- Automatically resizes

## Introduction to Classes

Classes are blueprints for creating objects with behavior:

```ahk
#Requires AutoHotkey v2.0

; Define a class
class Dog {
    ; Constructor - runs when creating new dog
    __New(name, breed) {
        this.name := name
        this.breed := breed
        this.energy := 100
    }

    ; Method - a function that belongs to the class
    Bark() {
        MsgBox this.name " says: Woof!"
    }

    Play() {
        this.energy -= 20
        MsgBox this.name " is playing! Energy: " this.energy
    }

    Rest() {
        this.energy := 100
        MsgBox this.name " is resting. Energy restored!"
    }
}

; Create instances (specific dogs)
myDog := Dog("Buddy", "Golden Retriever")
yourDog := Dog("Max", "Beagle")

; Call methods
myDog.Bark()       ; "Buddy says: Woof!"
myDog.Play()       ; Energy goes down
myDog.Rest()       ; Energy restored

yourDog.Bark()     ; "Max says: Woof!"
```

**Class concepts:**
- `class ClassName` defines a class
- `__New()` is the constructor
- `this` refers to the current instance
- Methods are functions inside the class
- Create instances with `ClassName(arguments)`

### Practical Class Example: Task Manager

```ahk
#Requires AutoHotkey v2.0

class Task {
    __New(description) {
        this.description := description
        this.completed := false
        this.createdAt := FormatTime(, "yyyy-MM-dd HH:mm")
    }

    Complete() {
        this.completed := true
        this.completedAt := FormatTime(, "yyyy-MM-dd HH:mm")
    }

    ToString() {
        status := this.completed ? "[✓]" : "[ ]"
        return status " " this.description
    }
}

class TaskList {
    __New() {
        this.tasks := []
    }

    Add(description) {
        task := Task(description)
        this.tasks.Push(task)
        return task
    }

    ShowAll() {
        if (this.tasks.Length = 0) {
            MsgBox "No tasks yet!"
            return
        }

        output := "Tasks:`n`n"
        for index, task in this.tasks {
            output .= index ". " task.ToString() "`n"
        }
        MsgBox output
    }

    CompleteTask(index) {
        if (index <= this.tasks.Length)
            this.tasks[index].Complete()
    }
}

; Usage
myTasks := TaskList()
myTasks.Add("Learn AutoHotkey")
myTasks.Add("Write a script")
myTasks.Add("Practice daily")

myTasks.ShowAll()            ; Shows all tasks
myTasks.CompleteTask(1)      ; Complete first task
myTasks.ShowAll()            ; Shows updated status
```

## Understanding Closures

Closures are functions that "remember" variables from where they were created:

```ahk
#Requires AutoHotkey v2.0

; Function that creates a counter
MakeCounter() {
    count := 0  ; This variable is "captured" by the closure

    increment() {
        count += 1
        return count
    }

    return increment  ; Return the function
}

; Create two independent counters
counter1 := MakeCounter()
counter2 := MakeCounter()

; Each counter has its own 'count' variable
MsgBox counter1()  ; 1
MsgBox counter1()  ; 2
MsgBox counter1()  ; 3

MsgBox counter2()  ; 1 (independent from counter1)
MsgBox counter2()  ; 2
```

**How closures work:**
- Inner function `increment()` can access outer variable `count`
- Each call to `MakeCounter()` creates a new, independent `count`
- The variable persists even after `MakeCounter()` returns

### Practical Closure Example: Event Handlers

Closures are perfect for GUI event handlers:

```ahk
#Requires AutoHotkey v2.0

CreateCounterGUI() {
    count := 0  ; This will be captured by closures

    gui := Gui()
    gui.Title := "Counter"

    countText := gui.Add("Text", "w200 Center", "Count: 0")

    ; These closures share the same 'count' variable
    gui.Add("Button", "w200", "Increment").OnEvent("Click", (*) => (
        count++,
        countText.Text := "Count: " count
    ))

    gui.Add("Button", "w200", "Decrement").OnEvent("Click", (*) => (
        count--,
        countText.Text := "Count: " count
    ))

    gui.Add("Button", "w200", "Reset").OnEvent("Click", (*) => (
        count := 0,
        countText.Text := "Count: 0"
    ))

    gui.Show()
}

CreateCounterGUI()
```

**Benefits:**
- No global variables needed
- Each GUI instance has its own state
- Code is clean and self-contained

## Error Handling with Try/Catch

Errors happen. Handle them gracefully:

```ahk
#Requires AutoHotkey v2.0

; Basic try/catch
try {
    result := 10 / 0  ; This will cause an error
    MsgBox result
} catch as err {
    MsgBox "Error occurred: " err.Message
}

MsgBox "Script continues!"
```

### Reading Files Safely

```ahk
#Requires AutoHotkey v2.0

ReadConfig(filename) {
    try {
        content := FileRead(filename)
        return content
    } catch OSError as err {
        MsgBox "Could not read file: " err.Message
        return ""  ; Return empty string on error
    }
}

; Usage
config := ReadConfig("settings.ini")
if (config = "")
    MsgBox "Using default settings"
else
    MsgBox "Config loaded successfully"
```

### Finally: Guaranteed Cleanup

The `finally` block always runs, even if there's an error:

```ahk
#Requires AutoHotkey v2.0

ProcessFile(filename) {
    file := FileOpen(filename, "r")

    try {
        data := file.Read()
        ; Process data...
    } catch as err {
        MsgBox "Error processing file: " err.Message
    } finally {
        file.Close()  ; ALWAYS closes file, even on error
    }
}
```

**Try/catch/finally structure:**
- `try` - code that might error
- `catch` - handles the error
- `finally` - always runs (cleanup)

## Working with Properties

Properties look like variables but can have custom logic:

```ahk
#Requires AutoHotkey v2.0

class Temperature {
    __New(celsius) {
        this._celsius := celsius
    }

    ; Property with getter and setter
    Celsius {
        get => this._celsius
        set => this._celsius := value
    }

    Fahrenheit {
        get => this._celsius * 9/5 + 32
        set => this._celsius := (value - 32) * 5/9
    }
}

; Usage
temp := Temperature(0)
MsgBox temp.Celsius     ; 0
MsgBox temp.Fahrenheit  ; 32

temp.Fahrenheit := 212  ; Set using Fahrenheit
MsgBox temp.Celsius     ; 100 (automatically converted)
```

**Property features:**
- `get =>` defines what happens when reading
- `set =>` defines what happens when writing
- Can validate or transform values
- Looks like a field but runs code

## Putting It All Together: Settings Manager

Let's combine everything into a practical example:

```ahk
#Requires AutoHotkey v2.0

class Settings {
    __New(filename := "settings.ini") {
        this.filename := filename
        this.data := Map()
        this.Load()
    }

    Load() {
        try {
            content := FileRead(this.filename)
            lines := StrSplit(content, "`n")

            for line in lines {
                line := Trim(line)
                if (line = "" || SubStr(line, 1, 1) = ";")
                    continue  ; Skip empty lines and comments

                parts := StrSplit(line, "=", , 2)
                if (parts.Length = 2) {
                    key := Trim(parts[1])
                    value := Trim(parts[2])
                    this.data[key] := value
                }
            }
        } catch OSError {
            ; File doesn't exist - use defaults
            this.SetDefaults()
        }
    }

    SetDefaults() {
        this.data["theme"] := "dark"
        this.data["fontSize"] := "12"
        this.data["autosave"] := "true"
    }

    Get(key, defaultValue := "") {
        return this.data.Has(key) ? this.data[key] : defaultValue
    }

    Set(key, value) {
        this.data[key] := value
    }

    Save() {
        content := "; Application Settings`n`n"
        for key, value in this.data {
            content .= key " = " value "`n"
        }

        try {
            FileDelete(this.filename)
        } catch {
            ; File might not exist
        }

        FileAppend(content, this.filename)
    }
}

; Usage example
settings := Settings("myapp.ini")

; Read settings
theme := settings.Get("theme", "light")
fontSize := settings.Get("fontSize", "10")

MsgBox "Theme: " theme "`nFont Size: " fontSize

; Modify settings
settings.Set("theme", "light")
settings.Set("fontSize", "14")

; Save to disk
settings.Save()

MsgBox "Settings saved!"
```

## Next Steps

You've learned:
- ✓ Objects, arrays, and maps
- ✓ Classes and object-oriented programming
- ✓ Closures and their practical uses
- ✓ Error handling with try/catch
- ✓ Properties with custom logic

**Continue learning:**
- docs/manuals/ahk2/explanation/closures-deep-dive.md for closure internals
- docs/manuals/ahk2/explanation/class-system.md for advanced OOP
- docs/manuals/ahk2/how-to/ for specific task solutions
- docs/manuals/ahk2/reference/ for complete language specification

## Practice Exercises

1. Create a `BankAccount` class with deposit/withdraw methods
2. Build a `Library` class that manages a collection of books
3. Write a function that uses closures to create a password validator
4. Create a GUI app with error handling for file operations
5. Build a class with properties that validate their values

## Common Intermediate Mistakes

**Mistake 1: Forgetting `this` in classes**
```ahk
class Wrong {
    __New(name) {
        name := name  ; Wrong - doesn't save to instance
    }
}

class Right {
    __New(name) {
        this.name := name  ; Right - saves to instance
    }
}
```

**Mistake 2: Array index starting at 0**
```ahk
arr := ["First", "Second"]
MsgBox arr[0]  ; Error! Arrays start at 1
MsgBox arr[1]  ; Correct - shows "First"
```

**Mistake 3: Not handling errors in file operations**
```ahk
; Wrong - will crash if file doesn't exist
content := FileRead("missing.txt")

; Right - handles missing file gracefully
try {
    content := FileRead("missing.txt")
} catch {
    content := "Default content"
}
```

**Mistake 4: Capturing loop variables in closures**
See docs/manuals/ahk2/how-to/closure-loop-pattern.md for detailed solution.

Congratulations! You've completed the intermediate tutorial. You're now ready to build sophisticated AutoHotkey applications.
