---
description: Create a new AHK2 window management feature with research-driven development
argument-hint: [feature-name-or-description]
---

# Create New AHK2 Feature

Build a new window management feature for A Movement Suite using research-driven development that consults the AHK2 manual before implementation.

## Instructions

You MUST delegate this task to the **ahk2-feature-builder** agent. The agent will:

1. Research the feature in `manuals/ahk2/` to understand technical challenges
2. Present a detailed research summary with:
   - Technical difficulties identified
   - Relevant patterns from the manual
   - Memory safety considerations
   - Implementation complexity estimate
3. Wait for your approval before implementing
4. Implement following project patterns and AHK2 best practices
5. Update all relevant documentation

## Invocation

Delegate to the agent with this exact format:

```
Use the ahk2-feature-builder agent to create: $ARGUMENTS

Make sure to:
- Search manuals/ahk2/ for relevant patterns first
- Present research summary before implementing
- Follow all memory safety guidelines
- Apply flicker prevention to any GUIs
- Update docs/features.md when done
```

## Usage Examples

**Basic usage:**
```
/feature window transparency slider
```

**Detailed specification:**
```
/feature Add a hotkey (LCtrl+LShift+MButton) that shows a popup slider to adjust window transparency from 0-255. Should have preview overlay and remember last used transparency per window.
```

**Hotkey addition:**
```
/feature RAlt+F11 should toggle window borders on/off for the active window
```

**Enhancement to existing feature:**
```
/feature Enhance WindowMove to support magnetic edges that "stick" when windows get close
```

## What the Agent Will Do

### Research Phase
- Search `manuals/ahk2/reference/` for technical details
- Read `manuals/ahk2/how-to/` for implementation patterns
- Check `manuals/ahk2/explanation/` for architectural guidance
- Review project docs: `docs/features.md`, `docs/claude/code-conventions.md`

### Planning Phase
- Present findings with manual references
- Identify potential issues (memory leaks, flicker, edge cases)
- Estimate complexity and risk
- Wait for approval

### Implementation Phase
- Create feature file in `src/Features/`
- Add toggle function to `src/lib/Core/ToggleUtils.ahk2`
- Update `AWindowMovementSuite.ahk2` with include and tray menu
- Apply memory safety patterns (timer cleanup, GUI destruction)
- Apply flicker prevention (`LockWindowUpdate`)

### Documentation Phase
- Update `docs/features.md` with complete feature documentation
- Update other docs if architecture changed
- Add memory considerations to checklist if new patterns

## Notes

- **Research is mandatory:** Agent will NOT skip manual research
- **Approval required:** Agent waits for your OK before implementing
- **Memory safe by default:** Agent applies all leak prevention patterns
- **Documentation included:** Not optional
- **Project patterns enforced:** Follows existing code style

## Related Commands

- `/plan` - General planning tool (not AHK2-specific)
- `/doc` - Update documentation after changes
- `/commit` - Create proper git commits

---

**The agent provides AHK2 expertise and ensures implementation quality. Use it for all new features!**
