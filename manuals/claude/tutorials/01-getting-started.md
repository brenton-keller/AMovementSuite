# Tutorial 01: Getting Started with Claude Code

**Time Estimate:** 30-60 minutes
**Prerequisites:**
- Claude Code installed (Node.js 18+)
- Basic understanding of version control (git)
- A project to work on (or create a test project)

**What You'll Learn:**
- Set up your first CLAUDE.md file
- Understand the three-layer configuration system
- Write effective project context
- Make your first Claude Code improvements

---

## Before You Begin: The User-Driven Philosophy

**Critical mindset shift:** Claude Code is a collaborative tool, not an automatic solution. You will:
- ✅ Gradually build up your configuration
- ✅ Add context as you discover what's needed
- ✅ Iterate based on real conversations
- ❌ NOT get an instant perfect setup
- ❌ NOT have everything configured automatically

This tutorial guides you through **your first steps** - not a complete system.

---

## Part 1: Understanding the Three Layers (5 minutes)

Claude Code operates through three complementary layers:

### Layer 1: CLAUDE.md Files (Project Memory)
- **What:** Markdown files that load automatically when you start Claude Code
- **Where:** `.claude/CLAUDE.md` or `CLAUDE.md` in your project root
- **Purpose:** Provide essential project context that applies to every conversation
- **Token cost:** Loaded once at session start

### Layer 2: Slash Commands (Reusable Workflows)
- **What:** Templated workflows stored in `.claude/commands/`
- **Purpose:** Extract repetitive tasks (testing, linting, formatting)
- **Activation:** You type `/command-name` when needed

### Layer 3: Subagents (Specialized Assistants)
- **What:** Isolated AI assistants with specific expertise in `.claude/agents/`
- **Purpose:** Handle specialized tasks with their own context
- **Activation:** Automatic delegation or explicit invocation

**For this tutorial, we focus on Layer 1: CLAUDE.md**

---

## Part 2: Create Your First CLAUDE.md (15 minutes)

### Step 1: Initialize with the Built-in Command

Navigate to your project directory and run:

```bash
cd your-project
claude
```

In Claude Code, type:
```
/init
```

This interactive command will:
- Detect your project type
- Generate a basic CLAUDE.md
- Create directory structure

**Review the output, don't just accept it.**

### Step 2: Understand What Was Generated

Open `.claude/CLAUDE.md` and look for these sections:

```markdown
# Project Name

## Tech Stack
[Languages, frameworks, tools]

## Key Commands
[How to build, test, run]

## Directory Structure
[Where things live]

## Coding Conventions
[Style preferences, patterns]
```

### Step 3: Refine for YOUR Project

The generated file is a starting point. **You** need to add project-specific knowledge.

**Example: Before (generic)**
```markdown
## Tech Stack
- Python 3.11
- pytest for testing
```

**Example: After (project-specific)**
```markdown
## Tech Stack
- Python 3.11+ required
- pandas, numpy for data manipulation
- pytest for testing
- black for formatting (run before commits)

## Data Conventions
- All timestamps must be UTC
- Column names use snake_case
- Missing values: use median imputation for numerical data
- Random seeds always set to 42 for reproducibility
```

### Step 4: Add Non-Obvious Information

**Bad:** General programming knowledge Claude already has
```markdown
## Conventions
- Use descriptive variable names
- Write comments for complex code
```

**Good:** Project-specific details Claude can't guess
```markdown
## Critical Constraints
- ❌ Never modify files in data/raw/ (immutable source)
- ❌ Don't use CSV for large datasets (use Parquet via pandas.read_parquet)
- ✅ All database queries go through src/db/connection.py (connection pooling)
- ✅ Statistical analysis requires assumption checks (see @docs/methods.md)
```

---

## Part 3: Keep It Concise (10 minutes)

### The 100-Line Guideline

**Goal:** Under 100 lines in your base CLAUDE.md

**Why:** Every line loads into every conversation. Focus on essentials.

### Progressive Disclosure with @ Syntax

For detailed information, use plain file path references (NOT @ syntax):

```markdown
## Architecture Decisions
See docs/architecture/decisions/ for ADRs

## Data Schemas
Column definitions: docs/data/schemas.md
Validation rules: docs/data/validation-rules.md

## API Documentation
OpenAPI spec: docs/api/openapi.yaml
```

**When Claude needs details, it will read those files as needed.**

### What Belongs in Base CLAUDE.md

**Include:**
- ✅ Tech stack and versions
- ✅ Development commands (build, test, run)
- ✅ Directory structure overview
- ✅ Critical constraints and conventions
- ✅ Data schemas (if data science project)
- ✅ Pointers to detailed docs

**Exclude:**
- ❌ General programming best practices
- ❌ Full API documentation (link it instead)
- ❌ Complete coding standards (summarize, then link)
- ❌ Detailed implementation guides

---

## Part 4: Test Your Configuration (15 minutes)

### Exercise 1: Ask a Project Question

Start a new Claude Code conversation:

```
What's the process for adding a new data source to this project?
```

**Observe:**
- Does Claude reference your CLAUDE.md?
- Does it follow your conventions?
- Does it know your directory structure?

**If "no" to any:** Your CLAUDE.md needs more context about data ingestion.

### Exercise 2: Request Code Following Conventions

```
Create a new Python module for data validation in the appropriate directory
```

**Check:**
- Did it put the file in the right location?
- Did it use your naming conventions?
- Did it include your required elements (type hints, docstrings)?

**Iterate:** Add missing conventions to CLAUDE.md and try again.

### Exercise 3: Test Constraint Awareness

```
I need to clean up some files in data/raw/
```

**Expected response:** Claude should warn you about the "never modify data/raw/" constraint.

**If it doesn't:** Make your constraints more prominent in CLAUDE.md:

```markdown
## Protected Areas (CRITICAL - READ FIRST)
- ❌ NEVER modify data/raw/* (immutable source data)
- ❌ NEVER commit credentials (.env files)
- ❌ NEVER skip tests before commits
```

---

## Part 5: Practical Patterns (10 minutes)

### Pattern 1: Development Commands

Make it easy for Claude to run your workflows:

```markdown
## Development Commands

### Setup
```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # or `venv\Scripts\activate` on Windows
pip install -r requirements.txt
```

### Testing
```bash
pytest tests/ --cov --cov-report=term-missing
```

### Formatting
```bash
black src/ tests/
```

### Running
```bash
python -m src.pipeline.run
```
```

### Pattern 2: Common Pitfalls Section

Document recurring issues:

```markdown
## Common Pitfalls

### Error: "SettingWithCopyWarning" in pandas
**Cause:** Modifying view instead of copy
**Solution:** Always use .copy() when subsetting:
```python
# Wrong
filtered = df[df['age'] > 25]
filtered['new_col'] = value

# Right
filtered = df[df['age'] > 25].copy()
filtered['new_col'] = value
```

### Error: "KeyError: 'customer_id'"
**Cause:** Missing expected column in input data
**Solution:** Check data schema: `df.info()`
```

### Pattern 3: Technology-Specific Context

For multi-language projects:

```markdown
## Language Responsibilities
- **Python:** Data engineering, ML models, API services
- **R:** Statistical analysis, reporting, Shiny dashboards
- **Shared data format:** Parquet files in data/processed/

## Cross-Language Conventions
- Use pathlib (Python) / here package (R) for file paths
- UTC timezone for all timestamps
- snake_case for variable names in both languages
```

---

## Part 6: Iteration Strategy (5 minutes)

### Start Small, Grow Gradually

**Week 1:** Basic CLAUDE.md
- Tech stack
- Key commands
- Directory structure

**Week 2:** Add as you go
- When Claude makes wrong assumption → Add clarification
- When you repeat instruction → Add to CLAUDE.md
- When error occurs → Add to common pitfalls

**Week 3:** Refine
- Remove unused sections
- Consolidate redundant info
- Add links to detailed docs

### The `/memory` Command

During conversations, when you want to remember something:

```
#Remember: All database connections must use connection pooling via src/db/connection.py
```

Claude Code automatically adds this to your CLAUDE.md (prefixed with `#`).

---

## Part 7: Version Control Integration (5 minutes)

### What to Commit

**Commit to git:**
- ✅ `.claude/CLAUDE.md` (team-wide context)
- ✅ `.claude/commands/` (shared workflows)
- ✅ `.claude/agents/` (team agents)

**Do NOT commit:**
- ❌ `.claude/settings.local.json` (personal settings)
- ❌ `.claude/outputs/` (AI-generated artifacts)

### Create .gitignore Entry

Add to your `.gitignore`:

```gitignore
# Claude Code
.claude/settings.local.json
.claude/outputs/
```

### Share with Team

Your CLAUDE.md is now team documentation:

```bash
git add .claude/CLAUDE.md
git commit -m "Add Claude Code project configuration"
git push
```

**Team members get consistent context automatically.**

---

## Common Questions

### Q: How much detail should I include?

**A:** Include enough that Claude doesn't make incorrect assumptions about YOUR project. Exclude general programming knowledge.

**Test:** If a new team member would need to know it, include it.

### Q: Should I document every function?

**A:** No. Link to API documentation instead:

```markdown
## API Documentation
Generated docs: docs/api/index.html (run `make docs`)
```

### Q: What if my project uses multiple languages?

**A:** Create hierarchical CLAUDE.md files (covered in Tutorial 02).

### Q: Can I have both CLAUDE.md and .claude/CLAUDE.md?

**A:** Yes. `.claude/CLAUDE.md` loads first, then `CLAUDE.md` in subdirectories.

---

## Next Steps

**Immediate:**
1. Create your CLAUDE.md using the patterns above
2. Test it with 3-5 real questions
3. Iterate based on Claude's responses

**This Week:**
1. Add to CLAUDE.md when you notice gaps
2. Document one common pitfall
3. Version control your configuration

**Next Tutorial:**
When you're ready to add commands and agents, proceed to **Tutorial 02: Progressive Enhancement**.

---

## Success Checklist

After completing this tutorial, you should have:

- [ ] `.claude/CLAUDE.md` created and customized
- [ ] Tech stack documented
- [ ] Key commands listed
- [ ] At least one critical constraint added
- [ ] Tested with 3+ questions
- [ ] Committed to version control
- [ ] Under 150 lines (ideally under 100)

**Remember:** This is a living document. You'll continue refining it as you use Claude Code.

---

## Key Takeaways

1. **CLAUDE.md is project memory** - loaded automatically, focus on essentials
2. **Start small, iterate** - don't try to document everything at once
3. **Project-specific over general** - what makes YOUR project unique
4. **Under 100 lines** - use links for detailed documentation
5. **Version control it** - team gets consistent context
6. **User-driven** - YOU build this gradually, not automatically

**You've now established the foundation for productive Claude Code usage.**
