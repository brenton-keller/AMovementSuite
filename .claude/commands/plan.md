# Description: Structured task planning with complexity-based breakdown

Creates detailed, actionable plans with complexity ratings and incremental build strategy.

**Usage:**
- `/plan add undo/redo system` - Plan new feature implementation
- `/plan refactor config management` - Plan refactoring task
- `/plan fix window flash bug` - Plan bug fix approach

---

## Planning Philosophy

**Start Simple, Build Incrementally:**
- Get the core concept working first
- Build in a structure that's easily modifiable
- Add complexity and customization after core validates
- Never try to do everything at once

**Complexity as a Signal:**
- If task rates 6+, break into multiple separate builds
- Each phase should be independently deliverable
- Validate each step before adding more

---

## Planning Template

```
## Task: [Name]

**Complexity:** [1-5] (6+ = break this into multiple builds)
**Type:** [Feature/Refactor/Fix/Enhancement]

---

### Phase 1: [Name]
**Complexity:** [1-5]

**Objectives:**
- What this phase accomplishes

**Steps:**
- [ ] Specific actionable step
- [ ] Another actionable step
- [ ] Final step

**Key Outputs:**
- Thing to check for progress #1
- Thing to check for progress #2

---

### Phase 2: [Name]
**Complexity:** [1-5]

[Same structure...]

---

### Progressive Enhancement

**Iteration 1 (Core):**
- Bare minimum to validate the concept
- Gets basic functionality working
- Establishes structure for expansion

**Iteration 2 (Refinement):**
- Add configuration/customization
- Handle edge cases
- Improve user experience

**Iteration 3+ (Optional Enhancements):**
- Advanced features if core proves useful
- Performance optimizations
- Integration with other systems

---

### Dependencies

**Must exist first:**
- Prerequisite 1

**May conflict with:**
- Potential conflict 1

---

### Testing Checkpoints

- [ ] Core functionality validated
- [ ] Edge cases handled
- [ ] Integration with existing features

---

**Proceed?** [Yes/Refine plan/Cancel]
```

---

## Complexity Scale

**1 - Trivial:**
- Single file, < 50 lines changed
- Clear path, no design decisions
- Zero dependencies

**2 - Simple:**
- 1-2 files modified
- Straightforward implementation
- Minimal dependencies

**3 - Medium:**
- Multiple files affected
- Some design decisions needed
- Moderate dependencies

**4 - Complex:**
- System-wide changes
- Architectural decisions required
- Many dependencies or unknowns

**5 - Very Complex:**
- Major feature/refactor
- Multiple subsystems affected
- Significant architectural impact

**6+ = Stop and Break Up:**
- If you estimate 6+, this is too large
- Break into multiple separate builds
- Each build should be ≤5 complexity

---

## Process

1. **Understand the task:**
   - What is being requested?
   - Read relevant code/docs to understand current state
   - Identify scope

2. **Assess complexity:**
   - How many components affected?
   - What unknowns exist?
   - Rate on 1-5 scale

3. **Break into phases:**
   - Each phase independently completable
   - Rate each phase complexity (1-5)
   - 2-5 phases for most tasks

4. **Plan progressive enhancement:**
   - Iteration 1: Core concept only
   - Iteration 2: Refinements and edge cases
   - Iteration 3+: Optional enhancements

5. **Identify dependencies:**
   - What must exist first?
   - What might conflict?

6. **Create actionable steps:**
   - Each step specific and testable
   - Checkbox format for tracking
   - Define key outputs to validate progress

7. **Present plan:**
   - Show complete plan
   - Offer to refine if needed
   - If 6+: Suggest breaking into multiple builds

---

## Example Plan

```
## Task: Add Undo/Redo System for Window Operations

**Complexity:** 4 (Complex - multi-phase feature with architectural impact)
**Type:** Feature

---

### Phase 1: Basic History Stack
**Complexity:** 2

**Objectives:**
- Create foundation for storing window states
- Validate core concept works

**Steps:**
- [ ] Add history array to GlobalConfig (simple implementation)
- [ ] Define WindowState object (hwnd, x, y, w, h)
- [ ] Implement basic push/pop functions
- [ ] Test with manual function calls (not hooked yet)

**Key Outputs:**
- History stack exists and can store/retrieve states
- Manual test proves concept works

---

### Phase 2: Integration with One Feature
**Complexity:** 3

**Objectives:**
- Hook into ONE window operation (WindowMove)
- Prove integration pattern works before scaling

**Steps:**
- [ ] Add SaveState() call at start of WindowMove
- [ ] Add Ctrl+Z hotkey that restores last state
- [ ] Test undo on window moves only
- [ ] Validate this approach before expanding

**Key Outputs:**
- Can undo window moves
- Integration pattern established
- Ready to replicate to other features

---

### Phase 3: Expand to All Operations
**Complexity:** 2

**Objectives:**
- Apply proven pattern to remaining features
- Add redo support

**Steps:**
- [ ] Hook WindowScaleWidth (copy pattern from Phase 2)
- [ ] Hook WindowScaleHeight
- [ ] Hook WindowScaleXY
- [ ] Add Ctrl+Shift+Z for redo
- [ ] Add currentIndex pointer for redo navigation

**Key Outputs:**
- All window operations support undo/redo
- Redo functionality works

---

### Phase 4: Polish & Documentation
**Complexity:** 1

**Steps:**
- [ ] Add visual feedback (tooltip: "Undid: Window Move")
- [ ] Update docs/features.md
- [ ] Test edge cases (window closed, etc.)

**Key Outputs:**
- User-facing polish complete
- Documentation updated

---

### Progressive Enhancement

**Iteration 1 (Core - Phases 1-2):**
- Just history stack + undo for WindowMove
- Validates concept before going further
- Establishes integration pattern
- **Stop here if approach doesn't work well**

**Iteration 2 (Full Feature - Phases 3-4):**
- Expand to all operations
- Add redo support
- Polish and document

**Iteration 3+ (Future Enhancements):**
- Per-window history (instead of global)
- Persistent history across sessions
- Undo preview (show where window will go)
- History viewer UI
- **Only if Iteration 1-2 proves valuable**

---

### Dependencies

**Must exist first:**
- WindowMove, WindowScale* features
- GlobalConfig system

**May conflict with:**
- Apps using Ctrl+Z (need context detection)
- Consider LCtrl+Z to avoid conflicts

---

### Testing Checkpoints

- [ ] Phase 1: Manual history push/pop works
- [ ] Phase 2: Undo works for WindowMove
- [ ] Phase 3: Undo/redo works for all operations
- [ ] Edge case: Window closed between save and restore
- [ ] Integration: Works with all window features

---

**Proceed?** [Yes - start with Iteration 1 / Refine plan / Cancel]
```

---

## Key Principles

1. **Complexity-driven:** Use 1-5 scale; 6+ means break it up
2. **Incremental:** Build core first, validate, then enhance
3. **Actionable:** Checkbox steps that are specific and testable
4. **Phased:** Each phase has its own complexity and outputs
5. **Validatable:** Key outputs let you verify progress
6. **Stop points:** Each iteration is a natural place to pause and evaluate

---

## Language Adaptation

Works across all languages by adapting structure:

**AHK:** Phases match modular feature structure
**Python:** Includes module/test/doc phases
**React:** Component/state/styling phases
**General:** Learns project patterns and adapts
