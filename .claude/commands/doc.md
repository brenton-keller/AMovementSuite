# Description: Documentation update assistant with project documentation philosophy

Ensures documentation stays synchronized with code changes and follows project standards.

**Usage:**
- `/doc` - Check what changed, guide through documentation updates
- `/doc [component]` - Focus on specific component (e.g., "WindowDimmer feature")

---

## Documentation Philosophy

When updating or creating documentation, follow these principles:

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
   - Links to related documentation
   - "See X for details on Y"
   - Build a web of knowledge

5. **Update Reminders** (for claude.md files only)
   - Checklists prompting doc updates with code changes
   - Helps future developers/AI maintain docs

### Style Guidelines:

- **Thorough but concise:** Aim for 80% brevity vs exhaustive
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

### Audience Hierarchy:

Different docs serve different purposes:

1. **`claude.md` files** (AI Context + Developer Onboarding)
   - High-level overview and navigation
   - "Where to find what" guide
   - Common patterns and workflows
   - Update reminders for code changes
   - ~100-200 lines max

2. **`docs/` directory** (Technical Reference)
   - Detailed technical information
   - Architecture, patterns, APIs
   - Complete feature documentation
   - Can be longer and more comprehensive

3. **`README.md`** (User-Facing)
   - Installation, usage, features
   - End-user perspective
   - Quick start guides

---

## Process

1. **Detect changes:**
   - Run `git diff` to see what code changed
   - If user specified component, focus on that
   - Identify which subsystem was modified (Features, Config, UI, etc.)

2. **Determine documentation impact:**
   Based on change type, suggest docs to update:

   **If Features/ modified:**
   - [ ] `docs/features.md` - Technical details of the feature
   - [ ] `claude.md` - If architecture/patterns changed
   - [ ] `README.md` - If user-facing behavior changed

   **If Config/ modified:**
   - [ ] `docs/configuration.md` - Document new settings
   - [ ] `claude.md` - If affects common patterns

   **If UI/ modified:**
   - [ ] `docs/ui-components.md` - GUI changes

   **If MouseHotkeys/ modified:**
   - [ ] `docs/mouse-hotkeys.md` - Hotkey reference
   - [ ] `src/MouseHotkeys/claude.md` - If patterns changed

   **If architecture/structure changed:**
   - [ ] `docs/architecture.md` - System structure
   - [ ] `claude.md` - Directory structure, patterns

3. **Guide update process:**
   - Show current doc content in relevant section
   - Suggest what to add/change based on code modifications
   - Follow documentation philosophy (structure, style, audience)
   - Ensure cross-references are updated

4. **Verify completeness:**
   - Check all relevant docs were updated
   - Verify cross-references are correct
   - Confirm examples still work

---

## Language Adaptation

This command adapts to any project language:

**Python:**
- Updates docstrings (Google/NumPy/Sphinx style)
- Updates `docs/` (Sphinx, MkDocs, etc.)
- Updates README, API reference

**JavaScript:**
- Updates JSDoc comments
- Updates README, API docs
- Framework-specific: Storybook, component docs

**General:**
- Detects existing doc structure and matches it
- Learns project conventions from existing docs
- Adapts to markdown, RST, or other formats

---

## Example Workflow

```
You: /doc
Claude: I see changes to src/Features/WindowScaleXY.ahk2

Added live resize system with motion detection. This requires updates:

☐ docs/features.md
  Section: "Window Scaling (WindowScaleXY)"
  Add: "Live Resize System" subsection
  Content: Explain continuous updates, motion detection, thresholds

☐ claude.md (optional)
  Check if "Common Patterns" section needs update
  New pattern: Timer-based window updates

☐ README.md (likely not needed)
  Feature behavior unchanged from user perspective

Proceed with docs/features.md update? [Yes/Skip/Show me current content first]
```

---

## Key Reminders

- **Update docs WITH code, not after** - Keep them synchronized
- **Follow the philosophy** - Structure, style, audience
- **Cross-reference liberally** - Build knowledge web
- **Examples over explanation** - Show, don't just tell
- **Concise but thorough** - Quality over quantity
