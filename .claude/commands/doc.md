# Description: Documentation update assistant with project documentation philosophy

Ensures documentation stays synchronized with code changes and follows project standards.

**Usage:**
- `/doc` - Check what changed, guide through documentation updates
- `/doc [component]` - Focus on specific component (e.g., "WindowDimmer feature")

---

## Documentation Structure

This project uses a hierarchical documentation system:

**CLAUDE.md** (~100 lines) - High-level navigation and project overview
**docs/claude/** - Focused guides for AI context and developers
- `architecture-overview.md` - System structure and navigation
- `code-conventions.md` - Coding patterns and standards
- `development-guide.md` - Development workflow
- `git-workflow.md` - Commit philosophy and git practices
- `best-practices.md` - Claude Code best practices reference

**docs/** - Comprehensive technical reference
- `architecture.md` - Detailed system architecture
- `features.md` - Complete feature specifications
- `configuration.md` - Configuration system details
- `mouse-hotkeys.md` - Hotkey reference table
- `ui-components.md` - GUI component documentation
- `virtual-desktops.md` - Virtual desktop integration
- `contributing.md` - Contribution guidelines
- `development.md` - Development setup

**src/claude.md** - Source code organization guide
**src/MouseHotkeys/claude.md** - Mouse hotkey system details

---

## Documentation Philosophy

### Structure - Every Document Should Have:

1. **Purpose/Overview** (1 paragraph)
   - What is this component/feature/module?
   - Why does it exist?

2. **Key Concepts** (2-4 main sections)
   - Organized with clear headers
   - Bullets for clarity, not walls of text
   - Focus on the important ideas

3. **Examples** (when helpful)
   - Concrete usage examples
   - Real-world scenarios
   - Not exhaustive API listings

4. **Cross-References**
   - Links to related documentation with `@` syntax
   - "See X for details on Y"
   - Build a web of knowledge

5. **Update Reminders** (for claude.md files only)
   - Checklists prompting doc updates with code changes
   - Helps future developers/AI maintain docs

### Style Guidelines:

- **Thorough but concise:** Aim for clarity over comprehensiveness
  - Explain enough to understand, not every detail
  - 2-4 main sections, each with focused subsections

- **Organized, not dense:**
  - Use headers to break up content
  - Bullets for key points
  - Code blocks for examples

- **WHAT and WHY, not HOW:**
  - Explain what the component does and why it exists
  - Code shows how it's implemented
  - Exception: Complex algorithms may need explanation

### Progressive Disclosure with @ Imports:

Use `@path/to/file.md` in documentation to reference detailed content:
```markdown
See @docs/claude/development-guide.md for workflow.
See @docs/features.md for technical details.
```

This keeps high-level docs concise while detailed information remains accessible.

---

## Process

1. **Detect changes:**
   - Run `git diff` to see what code changed
   - If user specified component, focus on that
   - Identify which subsystem was modified

2. **Determine documentation impact:**
   Based on change type, suggest docs to update:

   **If Features/ modified:**
   - [ ] `docs/features.md` - Technical details of the feature
   - [ ] `CLAUDE.md` - Only if architecture/patterns changed significantly
   - [ ] `docs/claude/architecture-overview.md` - If affects system structure
   - [ ] `README.md` - If user-facing behavior changed

   **If Config/ modified:**
   - [ ] `docs/configuration.md` - Document new settings
   - [ ] `docs/claude/code-conventions.md` - If affects common patterns

   **If UI/ modified:**
   - [ ] `docs/ui-components.md` - GUI changes
   - [ ] `docs/claude/development-guide.md` - If affects UI development workflow

   **If MouseHotkeys/ modified:**
   - [ ] `docs/mouse-hotkeys.md` - Hotkey reference table
   - [ ] `src/MouseHotkeys/claude.md` - If patterns or structure changed

   **If architecture/structure changed:**
   - [ ] `docs/architecture.md` - Detailed system structure
   - [ ] `docs/claude/architecture-overview.md` - High-level overview
   - [ ] `CLAUDE.md` - If directory structure changed
   - [ ] `src/claude.md` - If source organization changed

   **If git workflow or conventions changed:**
   - [ ] `docs/claude/git-workflow.md` - Commit philosophy
   - [ ] `docs/claude/code-conventions.md` - Coding standards

   **If adding new slash commands:**
   - [ ] `CLAUDE.md` - List of slash commands
   - [ ] `.claude/commands/[name].md` - Create/update command file

3. **Guide update process:**
   - Show current doc content in relevant section
   - Suggest what to add/change based on code modifications
   - Follow documentation philosophy (structure, style, audience)
   - Ensure cross-references use `@` syntax
   - Keep CLAUDE.md under 100 lines (use `@` imports if needed)

4. **Verify completeness:**
   - Check all relevant docs were updated
   - Verify cross-references are correct
   - Confirm examples still work
   - Ensure progressive disclosure is maintained

---

## Examples

### Example 1: Feature Added

```
You: /doc
Claude: I see changes to src/Features/WindowDimmer.ahk2

Added new WindowDimmer feature with opacity control. This requires updates:

☐ docs/features.md
  Section: "Additional Features"
  Add: "Window Dimmer (WindowDimmer)" subsection
  Content: Explain opacity control, hotkeys, configuration

☐ docs/claude/architecture-overview.md
  Section: "Features/ - Window Management Modules"
  Add: WindowDimmer.ahk2 to feature list with brief description

☐ README.md (if user-facing)
  Add WindowDimmer to features list if it affects end users

☐ CLAUDE.md
  Already references @docs/features.md, no change needed (good!)

Proceed with docs/features.md update? [Yes/Skip/Show current content]
```

### Example 2: Hotkey Pattern Changed

```
You: /doc MouseHotkeys
Claude: I see changes to src/MouseHotkeys/hk_basic.ahk2

Changed modifier detection pattern for consistency. This requires:

☐ src/MouseHotkeys/claude.md
  Section: "Common Patterns > Modifier Detection Pattern"
  Update: Code example to show new pattern

☐ docs/claude/code-conventions.md
  Section: "Common Patterns"
  Add reference: See @src/MouseHotkeys/claude.md for hotkey patterns

☐ docs/mouse-hotkeys.md
  Verify examples still match new pattern

Proceed? [Yes/No/Show me what changed]
```

---

## Key Reminders

- **Update docs WITH code, not after** - Keep them synchronized
- **Follow the philosophy** - Structure, style, audience hierarchy
- **Use `@` imports** - Keep high-level docs concise with progressive disclosure
- **Cross-reference liberally** - Build knowledge web
- **Examples over explanation** - Show, don't just tell
- **Keep CLAUDE.md ~100 lines** - Use `@` imports for details
