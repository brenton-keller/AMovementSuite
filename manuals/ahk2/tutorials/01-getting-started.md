# Tutorial: Getting Started with AutoHotkey v2

This tutorial guides you through your first steps with AutoHotkey v2, from installation to creating your first practical scripts.

## Prerequisites

- Windows operating system
- Basic familiarity with text editing
- No prior programming experience required

## Installing AutoHotkey v2

1. Download AutoHotkey v2 from the official website
2. Run the installer
3. Choose "Install for all users" or "Install for current user only"
4. Verify installation by creating a test file

## Your First Script: Hello World

Create a new text file called `hello.ahk`:

```ahk
#Requires AutoHotkey v2.0
MsgBox "Hello, World!"
```

Double-click the file to run it. You should see a message box appear.

**What's happening:**
- `#Requires AutoHotkey v2.0` ensures your script runs on v2
- `MsgBox` is a function that displays a message box
- Strings in v2 use double quotes `"`

## Creating Your First Hotkey

A hotkey triggers when you press a key combination. Create `first_hotkey.ahk`:

```ahk
#Requires AutoHotkey v2.0

; Win+N opens Notepad
#n::Run "notepad.exe"
```

**Understanding the syntax:**
- `#n::` means "Windows key + n"
- `Run` starts a program
- `;` begins a comment (ignored by AutoHotkey)

**Common modifier symbols:**
- `#` = Windows key
- `^` = Control
- `!` = Alt
- `+` = Shift

Try these variations:

```ahk
; Ctrl+Alt+C opens Calculator
^!c::Run "calc.exe"

; Win+E opens Explorer (replacing default)
#e::Run "explorer.exe"
```

## Working with Text

AutoHotkey excels at text manipulation. Create `text_expander.ahk`:

```ahk
#Requires AutoHotkey v2.0

; Type "btw" followed by space/tab/enter to expand
:*:btw::by the way

; Type your email address
:*:@em::your.email@example.com

; Multi-line expansion
:*:addr::
(
John Smith
123 Main Street
City, State 12345
)
```

**Hotstring syntax:**
- `:*:` means trigger immediately (no need for enter/tab)
- `::` separates the trigger from replacement text
- Use `()` for multi-line text

## Creating a Simple GUI

GUIs (Graphical User Interfaces) let you create windows and buttons:

```ahk
#Requires AutoHotkey v2.0

; Create a GUI window
myGui := Gui()
myGui.Title := "My First GUI"

; Add a text label
myGui.Add("Text", , "Enter your name:")

; Add an input box
nameInput := myGui.Add("Edit", "w200")

; Add a button
submitBtn := myGui.Add("Button", "Default w200", "Say Hello")
submitBtn.OnEvent("Click", SayHello)

; Show the window
myGui.Show()

; Button click handler
SayHello(*) {
    global nameInput
    name := nameInput.Value
    MsgBox "Hello, " name "!"
}
```

**Key concepts:**
- `Gui()` creates a new window
- `Add()` puts controls in the window
- `OnEvent()` responds to clicks
- `*` in function parameters means "accept any number of arguments"

## Variables and Basic Operations

Variables store data for later use:

```ahk
#Requires AutoHotkey v2.0

; Variables don't need declaration
myName := "Alice"
myAge := 25

; String concatenation (joining)
greeting := "Hello, my name is " myName

; Math operations
nextYear := myAge + 1
double := myAge * 2

; Display results
MsgBox greeting "`nNext year I'll be " nextYear
```

**Important v2 changes:**
- Use `:=` for assignment (not `=`)
- No need for `%` around variables in expressions
- `` `n `` is newline (backtick-n)

## Conditional Logic

Make decisions in your scripts:

```ahk
#Requires AutoHotkey v2.0

age := 18

if (age >= 18) {
    MsgBox "You are an adult"
} else {
    MsgBox "You are a minor"
}

; One-line if statement
if (age >= 21)
    MsgBox "You can drink in the US"
```

**Comparison operators:**
- `=` or `==` - equals
- `!=` - not equals
- `>` - greater than
- `<` - less than
- `>=` - greater than or equal
- `<=` - less than or equal

## Loops

Repeat actions multiple times:

```ahk
#Requires AutoHotkey v2.0

; Loop a specific number of times
Loop 5 {
    MsgBox "This is iteration " A_Index
}

; Loop through an array
fruits := ["Apple", "Banana", "Orange"]
for index, fruit in fruits {
    MsgBox "Fruit #" index ": " fruit
}

; While loop
count := 1
while (count <= 3) {
    MsgBox "Count: " count
    count++
}
```

**Built-in variables:**
- `A_Index` - current loop iteration number (starts at 1)
- Many more A_ variables available

## Functions

Organize code into reusable pieces:

```ahk
#Requires AutoHotkey v2.0

; Define a function
Greet(name) {
    return "Hello, " name "!"
}

; Call the function
message := Greet("Bob")
MsgBox message

; Function with default parameter
Multiply(x, y := 2) {
    return x * y
}

MsgBox Multiply(5)      ; Returns 10 (5 * 2)
MsgBox Multiply(5, 3)   ; Returns 15 (5 * 3)
```

## Practical Example: Window Manager

Let's combine concepts into a useful tool:

```ahk
#Requires AutoHotkey v2.0

; Win+Up: Maximize active window
#Up::WinMaximize "A"

; Win+Down: Minimize active window
#Down::WinMinimize "A"

; Win+Left: Move window to left half of screen
#Left::{
    WinGetPos &x, &y, &w, &h, "A"
    screenWidth := A_ScreenWidth
    screenHeight := A_ScreenHeight
    WinMove 0, 0, screenWidth//2, screenHeight, "A"
}

; Win+Right: Move window to right half of screen
#Right::{
    WinGetPos &x, &y, &w, &h, "A"
    screenWidth := A_ScreenWidth
    screenHeight := A_ScreenHeight
    WinMove screenWidth//2, 0, screenWidth//2, screenHeight, "A"
}
```

**New concepts:**
- `"A"` means "active window"
- `&variable` passes by reference (variable gets modified)
- `//` is integer division
- `{}` creates a code block for multi-line hotkeys

## Next Steps

You've learned:
- ✓ Basic syntax and structure
- ✓ Hotkeys and hotstrings
- ✓ Variables and operations
- ✓ Conditional logic and loops
- ✓ Functions
- ✓ Simple GUIs
- ✓ Window manipulation

**Continue learning:**
- docs/manuals/ahk2/tutorials/02-intermediate-concepts.md for classes and objects
- docs/manuals/ahk2/how-to/ for specific task solutions
- docs/manuals/ahk2/reference/ for detailed language reference
- docs/manuals/ahk2/explanation/ for deep dives into advanced topics

## Common Beginner Mistakes

**Mistake 1: Forgetting #Requires**
Always start scripts with `#Requires AutoHotkey v2.0` to avoid v1/v2 confusion.

**Mistake 2: Using = instead of :=**
```ahk
; Wrong (v1 syntax)
x = 5

; Right (v2 syntax)
x := 5
```

**Mistake 3: Forgetting quotes around strings**
```ahk
; Wrong
MsgBox Hello

; Right
MsgBox "Hello"
```

**Mistake 4: Using %variable% in expressions**
```ahk
; Wrong (v1 syntax)
MsgBox % "Hello " %name%

; Right (v2 syntax)
MsgBox "Hello " name
```

## Troubleshooting

**Script won't run:**
- Check if AutoHotkey v2 is installed
- Verify file has `.ahk` extension
- Look for syntax errors (missing quotes, parentheses)

**Hotkey doesn't work:**
- Make sure no other program uses the same hotkey
- Try running script as administrator
- Check if hotkey syntax is correct

**Getting error messages:**
- Read the error carefully - it tells you the line number
- Check for missing parentheses or quotes
- Verify variable names are spelled correctly

## Practice Exercises

1. Create a hotkey that opens your favorite website
2. Make a hotstring that expands your full address
3. Create a GUI with two buttons: one opens Calculator, one opens Notepad
4. Write a function that converts Celsius to Fahrenheit
5. Make a script that shows the time when you press Ctrl+T

## Resources

- Official documentation: https://www.autohotkey.com/docs/v2/
- Community forum: https://www.autohotkey.com/boards/
- Example scripts in AutoHotkey installation folder

Congratulations! You've completed the getting started tutorial. You now have the foundation to create practical AutoHotkey scripts.
