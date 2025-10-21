# Contributing to A Movement Suite

Thank you for considering contributing to A Movement Suite! This document provides guidelines and workflows for contributing to the project.

## Table of Contents
1. [Getting Started](#getting-started)
2. [Development Workflow](#development-workflow)
3. [Code Standards](#code-standards)
4. [Documentation Requirements](#documentation-requirements)
5. [Testing Guidelines](#testing-guidelines)
6. [Pull Request Process](#pull-request-process)
7. [Issue Reporting](#issue-reporting)

---

## Getting Started

### Prerequisites
- Windows 10 or Windows 11
- AutoHotkey v2.0 or later ([Download](https://www.autohotkey.com/))
- Git for version control
- Text editor (VS Code, Sublime Text, etc.)

### Setup Development Environment

1. **Fork the repository**
   ```bash
   # Fork on GitHub, then clone your fork
   git clone https://github.com/YOUR_USERNAME/AMovementSuite.git
   cd AMovementSuite
   ```

2. **Install AutoHotkey v2.0**
   - Download from [autohotkey.com](https://www.autohotkey.com/)
   - Ensure `AutoHotkey64.exe` is in your PATH

3. **Run the application**
   ```bash
   # Right-click AWindowMovementSuite.ahk2 → Run Script
   # Or double-click the file
   ```

4. **Read the documentation**
   - Start with `claude.md` for project overview
   - Read `src/claude.md` for source structure
   - Check `docs/` for detailed references

---

## Development Workflow

### 1. Create a Feature Branch

```bash
git checkout -b feature/your-feature-name
# or
git checkout -b fix/issue-description
```

Branch naming conventions:
- `feature/` - New features
- `fix/` - Bug fixes
- `refactor/` - Code refactoring
- `docs/` - Documentation updates

### 2. Make Your Changes

Follow the guidelines in this document and:
- Maintain existing code style
- Add comments for complex logic
- Update documentation as you go

### 3. Test Thoroughly

See [Testing Guidelines](#testing-guidelines) below.

### 4. Commit Your Changes

```bash
git add .
git commit -m "Brief description of changes"
```

Commit message format:
```
<type>: <short summary>

<detailed description if needed>

Related issues: #123
```

Types: `feat`, `fix`, `docs`, `refactor`, `test`, `chore`

### 5. Push and Create Pull Request

```bash
git push origin feature/your-feature-name
```

Then create a Pull Request on GitHub.

---

## Code Standards

### AutoHotkey v2.0 Conventions

#### Naming Conventions
```ahk
; Functions: PascalCase
MoveWindowToPosition(x, y) {
    ; ...
}

; Variables: camelCase
windowID := WinGetID("A")
snapDistance := 10

; Constants: UPPER_CASE or PascalCase
SNAP_DISTANCE := 10
MinWindowWidth := 100

; Classes: PascalCase
class WindowManager {
    ; ...
}
```

#### Code Formatting
```ahk
; Braces on same line for functions and hotkeys
MyFunction() {
    ; Code here
}

Hotkey::{
    ; Code here
}

; Indentation: 4 spaces (not tabs)
if (condition) {
    DoSomething()
    DoSomethingElse()
}

; Spaces around operators
x := 10 + 20
result := (a > b) ? a : b

; No space before function parentheses
MyFunction(param1, param2)
```

#### Comments
```ahk
; Single-line comments for brief explanations
snapDistance := 10  ; Default snap distance in pixels

; Multi-line comments for complex logic
/*
    This section handles window snapping by comparing the edges of the
    moving window against all other visible windows and monitor boundaries.
    Snapping occurs when within snapDistance pixels.
*/
```

### File Organization

```ahk
; File header (optional but recommended)
; ==============================================================================
; WindowMove.ahk2
;
; Description: Window movement with intelligent edge snapping
; Author: Your Name
; License: MIT
; ==============================================================================

; Global variables at the top
global windowMoveEnabled := true
global snapDistance := 10

; Helper functions
IsValidWindow(winID) {
    ; ...
}

; Main functionality
Hotkey::{
    ; ...
}

; Toggle function
ToggleFeature(*) {
    ; ...
}
```

---

## Documentation Requirements

### When Adding Features

**You MUST update the following documentation:**

1. **`docs/features.md`**
   - Add feature to overview table
   - Create detailed feature section
   - Document hotkeys, configuration options
   - Provide usage examples

2. **`README.md`** (if user-facing)
   - Add to feature list
   - Update usage instructions
   - Add to known issues if applicable

3. **`claude.md`** (if architectural changes)
   - Update directory structure if new files
   - Update quick reference links

4. **Inline code comments**
   - Document non-obvious logic
   - Explain complex algorithms
   - Note gotchas and edge cases

### When Adding Mouse Hotkeys

**You MUST update:**

1. **`docs/mouse-hotkeys.md`**
   - Add to hotkey reference table
   - Document all modifier combinations
   - List context-specific behaviors
   - Provide usage examples

2. **`src/MouseHotkeys/claude.md`**
   - Update if adding new patterns or file

### When Modifying Configuration

**You MUST update:**

1. **`docs/configuration.md`**
   - Document new config options
   - Specify defaults and valid ranges
   - Show usage examples

2. **`src/lib/Config/GlobalConfig.ahk2`**
   - Add default values with comments

### Documentation Style

- **Be concise**: High-level in claude.md files, detailed in docs/
- **Provide examples**: Show actual code usage
- **Link related docs**: Cross-reference with "See also" sections
- **Keep current**: Update docs in the same commit as code changes

---

## Testing Guidelines

### Manual Testing Checklist

#### For Window Features
- [ ] Test with various window types (browser, explorer, notepad, etc.)
- [ ] Test with maximized windows
- [ ] Test with minimized windows
- [ ] Test on primary monitor
- [ ] Test on secondary monitor (if available)
- [ ] Test with different window sizes (small, medium, large)
- [ ] Test edge cases (minimum size, screen boundaries)

#### For Mouse Hotkeys
- [ ] Test without modifiers
- [ ] Test with Ctrl modifier
- [ ] Test with Shift modifier
- [ ] Test with Alt modifier
- [ ] Test in target applications (browsers, code editors, etc.)
- [ ] Test in non-target applications (fallback behavior)
- [ ] Verify no conflicts with existing hotkeys

#### For Configuration Changes
- [ ] Test with default values
- [ ] Test with custom values
- [ ] Test persistence (restart application)
- [ ] Test invalid inputs (validation)
- [ ] Test settings GUI (if applicable)

### Testing Process

1. **Before testing**: Close other AutoHotkey scripts to avoid conflicts
2. **During testing**: Note any unexpected behavior or errors
3. **After testing**: Verify no side effects on other features

### Test Scenarios

**Example for WindowMove feature:**
```
Scenario: Move window with snapping enabled
1. Launch application
2. Open a browser window
3. Open a notepad window
4. Hold LAlt + RButton on browser
5. Move near notepad edge
6. Verify snapping occurs (ToolTip shows "Snapped to window edge")
7. Release mouse
8. Verify window positioned correctly
```

---

## Pull Request Process

### Before Submitting

1. **Test your changes thoroughly**
   - Follow testing guidelines above
   - Verify no regressions in existing features

2. **Update documentation**
   - See [Documentation Requirements](#documentation-requirements)

3. **Review your code**
   - Remove debug code (MsgBox, ToolTip for testing)
   - Check for code style consistency
   - Ensure comments are clear and helpful

4. **Clean commit history**
   - Consider squashing minor commits
   - Ensure commit messages are descriptive

### PR Description Template

```markdown
## Description
Brief description of what this PR does.

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Refactoring
- [ ] Documentation update

## Testing
- [ ] Tested on Windows 10
- [ ] Tested on Windows 11
- [ ] Tested with multiple window types
- [ ] Tested on multi-monitor setup

## Related Issues
Closes #123

## Screenshots (if applicable)
[Add screenshots or GIFs demonstrating the feature]

## Documentation
- [ ] Updated docs/features.md
- [ ] Updated README.md
- [ ] Updated inline comments
- [ ] Updated claude.md files
```

### Review Process

1. Maintainer reviews code and documentation
2. Automated checks (if configured)
3. Request changes if needed
4. Approval and merge

---

## Issue Reporting

### Before Opening an Issue

1. **Search existing issues** - Your issue may already be reported
2. **Check known issues** - See README.md Known Issues section
3. **Test with latest version** - Update and retry

### Issue Template

**Bug Report:**
```markdown
**Describe the bug**
Clear description of what's wrong.

**To Reproduce**
Steps to reproduce:
1. Do this
2. Do that
3. See error

**Expected behavior**
What you expected to happen.

**Actual behavior**
What actually happened.

**Environment**
- Windows Version: [e.g., Windows 11 22H2]
- AutoHotkey Version: [e.g., v2.0.11]
- A Movement Suite Version: [e.g., commit hash or release tag]

**Screenshots**
If applicable, add screenshots.

**Additional context**
Any other relevant information.
```

**Feature Request:**
```markdown
**Is your feature request related to a problem?**
Description of the problem.

**Describe the solution you'd like**
Clear description of desired functionality.

**Describe alternatives considered**
Other approaches you've considered.

**Additional context**
Screenshots, mockups, or examples.
```

---

## Code Review Checklist

### For Reviewers

- [ ] Code follows style guidelines
- [ ] Changes are well-documented
- [ ] No obvious bugs or issues
- [ ] Performance considerations addressed
- [ ] Error handling is appropriate
- [ ] Multi-monitor scenarios considered
- [ ] Documentation updated correctly
- [ ] No debug code left in

### For Contributors

Before requesting review:
- [ ] Self-review your code
- [ ] Test thoroughly
- [ ] Update all relevant documentation
- [ ] Clean up commit history
- [ ] Respond to PR feedback promptly

---

## Communication

### Where to Ask Questions

- **GitHub Issues** - Bug reports, feature requests
- **GitHub Discussions** - General questions, ideas
- **Pull Request Comments** - Code-specific discussions

### Be Respectful

- Be patient with reviewers
- Provide constructive feedback
- Assume good intentions
- Help others when you can

---

## Recognition

Contributors will be:
- Listed in project contributors
- Credited in release notes
- Acknowledged in documentation (where appropriate)

---

## License

By contributing to A Movement Suite, you agree that your contributions will be licensed under the MIT License.

---

**Thank you for contributing to A Movement Suite!**

For more information:
- Project Overview: `claude.md`
- Architecture: `docs/architecture.md`
- Development Setup: `docs/development.md`
