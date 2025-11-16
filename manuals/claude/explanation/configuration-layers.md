# Configuration Layers: The Three-Tier Context System

## Overview

Claude Code operates through three distinct configuration layers: CLAUDE.md files for persistent project memory, slash commands for reusable workflows, and subagents for specialized expertise. Understanding when and why to use each layer transforms Claude Code from a helpful assistant into a force multiplier for development.

## The Fundamental Problem: Context Without Repetition

Every conversation with Claude Code starts with zero project knowledge. Without configuration layers, you would need to explain:
- Your tech stack and conventions
- Project structure and key files
- Development commands and workflows
- Domain-specific requirements
- Critical constraints and rules

Repeating this information wastes time and introduces inconsistency. The three-layer system solves this through strategic context management.

## Layer 1: CLAUDE.md - Automatic Project Memory

### What It Is

CLAUDE.md is a markdown file that loads automatically when Claude Code starts in your project. It serves as persistent project memory, providing essential context without manual intervention.

### When to Use CLAUDE.md

Use CLAUDE.md for information that:
1. **Applies to all interactions** with this project
2. **Rarely changes** (stable project fundamentals)
3. **Is project-specific** (not general programming knowledge)
4. **Must always be remembered** (critical constraints, standards)

### What Belongs in CLAUDE.md

**Essential inclusions:**
- Tech stack and versions
- Key development commands
- Directory structure overview
- Coding standards and conventions
- Critical constraints (what never to do)
- Data schemas (for data projects)

**Example - Data Science Project:**
```markdown
# Analytics Platform

## Stack
Python 3.11, pandas, scikit-learn, pytest

## Commands
- Tests: `pytest tests/`
- Notebooks: `jupyter lab`
- Pipeline: `python -m src.pipeline.run`

## Data Conventions
- Raw data in data/raw/ (never modify)
- Processed data as Parquet in data/processed/
- Column naming: snake_case
- Timestamps: UTC only

## Code Standards
- Type hints required on all functions
- Median imputation for missing numerical values
- Bonferroni correction for multiple comparisons
- 80% test coverage minimum

## Protected Areas
- Never modify data/raw/ (immutable)
- Never remove tests to make them pass
- Never hardcode credentials
```

**Non-essentials (belongs elsewhere):**
- Detailed tutorials (too verbose)
- API documentation (reference material)
- Implementation details (code comments better)
- Changelog information (separate file)

### Why 100 Lines is the Target

Every line in CLAUDE.md consumes context tokens in EVERY interaction. A 1000-line CLAUDE.md means every simple question costs 1000+ tokens of context before Claude even reads your query.

**Token economics:**
- Average CLAUDE.md line: ~10 tokens
- 100-line CLAUDE.md: ~1000 tokens per interaction
- 1000-line CLAUDE.md: ~10,000 tokens per interaction
- At 10 interactions/hour: 10,000 vs 100,000 tokens

Conciseness isn't just aesthetic - it's economical and performant.

### The Hierarchy: Enterprise, User, Project

CLAUDE.md files can exist at three levels:

**Enterprise Level** (company-wide):
```
~/.config/claude/CLAUDE.md
```
Contains organizational policies:
- Security requirements
- Approved libraries
- Compliance rules
- Company coding standards

**User Level** (your preferences):
```
~/.claude/CLAUDE.md
```
Contains personal preferences:
- Your coding style
- Preferred patterns
- Shortcuts you like

**Project Level** (team conventions):
```
/project/CLAUDE.md
/project/python/CLAUDE.md
/project/r/CLAUDE.md
```
Contains project-specific context.

**Resolution order:** Enterprise → User → Project (most specific wins)

This hierarchy enables:
- Organizations to enforce standards
- Individuals to customize workflow
- Projects to override when needed

### Progressive Disclosure Through Subdirectories

Instead of one massive root CLAUDE.md, create contextual layers:

```
project/
├── CLAUDE.md (100 lines: project-wide essentials)
├── python/
│   └── CLAUDE.md (Python-specific conventions)
├── r/
│   └── CLAUDE.md (R-specific conventions)
└── shared/
    └── CLAUDE.md (Cross-language patterns)
```

When Claude Code navigates into `python/`, it loads:
1. Root CLAUDE.md (always)
2. python/CLAUDE.md (contextual)

This provides relevant context without overwhelming the base layer.

## Layer 2: Slash Commands - Reusable Workflows

### What They Are

Slash commands are markdown files in `.claude/commands/` that define reusable workflows. They extract repetitive multi-step processes into templates that accept arguments.

### When to Use Commands

Use commands for workflows that:
1. **Repeat regularly** (at least weekly)
2. **Follow consistent steps** (predictable structure)
3. **Benefit from parameterization** (vary by input)
4. **Span multiple operations** (multi-step procedures)

### What Belongs in Commands

**Good command candidates:**
- Testing workflows (run tests, generate coverage, analyze failures)
- Code quality checks (lint, format, type-check)
- Data validation (schema checks, quality reports)
- Documentation generation (API docs, reports)
- Deployment procedures (build, test, deploy)

**Poor command candidates:**
- One-time operations (just do it)
- Highly variable workflows (too unpredictable)
- Simple single operations (overhead not justified)

### Anatomy of a Command

```markdown
---
description: Run comprehensive test suite with coverage
allowed-tools: Bash, Read
argument-hint: [test-path] [options]
---

# Run Tests

Execute tests for current project with coverage reporting.

## Auto-Detection
1. Check for pytest.ini → Run Python tests
2. Check for testthat/ → Run R tests
3. Check for jest.config.js → Run JavaScript tests

## Execution
[Detailed instructions with $ARGUMENTS placeholder]

## Reporting
Display summary, failures, coverage stats
```

**Key elements:**
- **Frontmatter:** Metadata about the command
- **Description:** What the command does (aids discoverability)
- **Instructions:** Step-by-step procedure for Claude to follow
- **Placeholders:** `$ARGUMENTS` or `$1`, `$2` for parameters

### Commands vs CLAUDE.md

**When context is needed every time:** CLAUDE.md
**When action is needed on demand:** Command

Example:
- "Always use type hints" → CLAUDE.md
- "Run linting and fix issues" → Command `/lint`

### Command Organization and Namespaces

Organize commands in subdirectories for clarity:

```
.claude/commands/
├── data/
│   ├── validate.md
│   ├── profile.md
│   └── clean.md
├── ml/
│   ├── train.md
│   └── evaluate.md
└── shared/
    ├── test.md
    └── format.md
```

Users invoke with `/validate` (not `/data-validate`), but organization aids discovery and maintenance.

### Cross-Language Portability

Well-designed commands detect context and adapt:

```markdown
# Test Command

## Detect Project Type
- Python? Run pytest
- R? Run testthat
- JavaScript? Run Jest
- Go? Run go test

## Execute Appropriate Command
[Language-specific implementations]
```

This pattern enables one command to serve multiple ecosystems.

## Layer 3: Subagents - Specialized Expertise

### What They Are

Subagents are separate AI assistants with isolated context windows and specialized system prompts. Each subagent has expertise in a specific domain and maintains its own conversation history.

### When to Use Subagents

Use subagents when:
1. **Expertise is specialized** (distinct domain knowledge)
2. **Context isolation is valuable** (prevent main thread pollution)
3. **Different models are appropriate** (speed vs capability trade-offs)
4. **Parallel work is beneficial** (multiple tasks simultaneously)

### What Defines a Good Subagent

**Strong subagent candidates:**
- Data quality checker (specialized validation logic)
- Statistical analyst (rigorous methodology)
- Code reviewer (consistent review standards)
- Documentation generator (style and format expertise)

**Poor subagent candidates:**
- General helper (no specialization)
- Rarely used tasks (overhead not justified)
- Simple operations (context isolation unnecessary)

### Anatomy of a Subagent

```markdown
---
name: statistical-analyst
description: Expert statistician for hypothesis testing and experimental
  design. MUST BE USED for statistical analysis, A/B tests, and inferential
  statistics.
tools: Read, NotebookRead, Bash, Write
model: sonnet
---

You are a senior statistical analyst with expertise in:
- Hypothesis testing and power analysis
- Experimental design
- Regression modeling
- Causal inference

## Workflow
1. Understand research question
2. Check data quality and assumptions
3. Select appropriate methods
4. Conduct analysis with diagnostics
5. Interpret results with limitations

## Best Practices
- Always check assumptions before tests
- Report effect sizes alongside p-values
- Use multiple comparison corrections
- Document methodological decisions
```

**Key elements:**
- **Name:** Identifier for invocation
- **Description:** When to delegate (aids automatic routing)
- **Tools:** Which operations this agent can perform
- **Model:** Haiku (fast) vs Sonnet (capable) vs Opus (exceptional)
- **System prompt:** Expertise definition and procedures

### Subagents vs Commands

**Commands:** Procedural workflows (how to do something)
**Subagents:** Expertise and judgment (what to do and why)

Example:
- Running tests: Command (procedural)
- Reviewing test quality: Subagent (judgment)

### Context Isolation Benefits

Main conversation with bloated context:
```
[Main thread with 10,000 tokens of context]
User: "Also, validate this dataset"
Claude: [Processes with full context overhead]
```

Delegated to subagent:
```
[Main thread: 5,000 tokens]
User: "Validate this dataset"
→ Delegates to data-quality-checker subagent
  [Fresh context: just dataset + validation rules]
  [Focused, efficient processing]
  [Returns results to main thread]
```

Benefits:
- **Efficiency:** Subagent works with minimal context
- **Focus:** No distraction from unrelated main thread
- **Quality:** Specialized expertise encoded in system prompt

### Model Selection Strategy

Different subagents can use different models:

**Haiku (fast, cheap):**
- Data validation (rule-based)
- Code formatting (deterministic)
- Simple transformations

**Sonnet (balanced):**
- Code review (moderate complexity)
- Documentation generation
- Statistical analysis

**Opus (exceptional, expensive):**
- Architectural decisions
- Complex debugging
- Research synthesis

This enables cost-effective delegation: use cheap models for simple tasks, powerful models only when justified.

## How the Three Layers Work Together

### Complementary Roles

**CLAUDE.md provides:** Static project knowledge
**Commands provide:** Reusable workflows
**Subagents provide:** Specialized expertise

### Interaction Patterns

**Pattern 1: Context + Command**
```
[CLAUDE.md loaded: Python project conventions]
User: /test
[Command executes with context awareness]
```

**Pattern 2: Context + Subagent**
```
[CLAUDE.md loaded: Data schema definitions]
User: "Check data quality"
[Delegates to data-quality-checker subagent]
[Subagent receives CLAUDE.md context + dataset]
```

**Pattern 3: Command → Subagent**
```
User: /code-review
[Command workflow:]
  1. Run linting
  2. Delegate to code-reviewer subagent
  3. Compile results
```

### Evolution Path

Most projects evolve through these stages:

**Stage 1: Just CLAUDE.md**
```
CLAUDE.md with essential project context
Manual commands as needed
```

**Stage 2: Add Commands**
```
CLAUDE.md (unchanged)
Commands for frequent workflows
Still manual invocation
```

**Stage 3: Add Subagents**
```
CLAUDE.md (unchanged)
Commands (unchanged)
Subagents for specialized tasks
Some automatic delegation
```

**Stage 4: Integrated System**
```
Hierarchical CLAUDE.md
Command library
Multiple specialized subagents
Workflows orchestrating all three
```

## Decision Framework: Which Layer?

Use this decision tree:

```
Does this information apply to EVERY interaction?
│
├─ YES → CLAUDE.md
│
└─ NO → Is this a multi-step workflow I repeat?
         │
         ├─ YES → Slash Command
         │
         └─ NO → Does this need specialized expertise?
                  │
                  ├─ YES → Subagent
                  │
                  └─ NO → Handle ad-hoc
```

### Specific Scenarios

**Scenario: Type hints required**
→ CLAUDE.md (always applies)

**Scenario: Run full test suite with coverage**
→ Command (repeatable workflow)

**Scenario: Review statistical methodology**
→ Subagent (specialized expertise)

**Scenario: One-time data migration**
→ Ad-hoc (not worth infrastructure)

## Common Patterns and Anti-Patterns

### Pattern: Minimal CLAUDE.md + Rich Commands

**Good:**
```markdown
# CLAUDE.md (50 lines)
Core tech stack
Key commands
Critical constraints

# Commands (10+ files)
Detailed workflows for common tasks
```

This keeps context lean while enabling sophisticated workflows.

### Anti-Pattern: Encyclopedia CLAUDE.md

**Bad:**
```markdown
# CLAUDE.md (2000+ lines)
Everything about everything
No commands needed because it's all here
```

**Problem:** Massive context overhead every interaction

### Pattern: Specialized Subagents with Clear Boundaries

**Good:**
```
data-quality-checker: Data validation only
statistical-analyst: Hypothesis testing only
ml-engineer: Model training only
```

**Bad:**
```
general-helper: Does everything
smart-agent: Also does everything
```

Clear specialization enables effective delegation.

### Anti-Pattern: Redundant Information Across Layers

**Bad:**
```markdown
# CLAUDE.md
Run tests with: pytest tests/ --cov

# .claude/commands/test.md
Run tests with: pytest tests/ --cov
```

**Good:**
```markdown
# CLAUDE.md
Test framework: pytest

# .claude/commands/test.md
Run pytest with coverage reporting and failure analysis
```

CLAUDE.md provides context; command provides workflow.

## Practical Implementation Guide

### Week 1: Create Basic CLAUDE.md
```markdown
1. Document tech stack
2. List key commands
3. Define critical constraints
4. Keep under 100 lines
```

### Week 2: Extract First Command
```markdown
1. Identify most repeated workflow
2. Create command file
3. Test with variations
4. Refine based on results
```

### Week 3: Create First Subagent
```markdown
1. Identify task requiring specialized expertise
2. Write subagent definition
3. Test explicit invocation
4. Enable automatic delegation
```

### Week 4: Refine and Expand
```markdown
1. Monitor which context is referenced most
2. Prune unused CLAUDE.md sections
3. Add commands for new repetitive tasks
4. Create subagents as specializations emerge
```

## Debunked Misconceptions

### Misconception 1: "CLAUDE.md should be comprehensive"
**Reality:** CLAUDE.md should be minimal. Comprehensive context belongs in reference docs that can be read on demand.

### Misconception 2: "Commands are just saved prompts"
**Reality:** Commands are procedural workflows with parameterization, tool permissions, and structured outputs. They're infrastructure, not snippets.

### Misconception 3: "Subagents are for large projects only"
**Reality:** Even solo developers benefit from specialized subagents for distinct expertise domains. Size doesn't matter; specialization does.

### Misconception 4: "More layers is better"
**Reality:** Each layer has overhead. Use the minimum necessary. Many projects thrive with just CLAUDE.md and a few commands.

### Misconception 5: "All three layers must be used"
**Reality:** Start with CLAUDE.md, add commands when patterns emerge, add subagents when specialization is justified. Incremental adoption prevents premature complexity.

## Conclusion

The three-layer configuration system exists because different needs require different mechanisms:

**Persistent knowledge** → CLAUDE.md
**Reusable procedures** → Commands
**Specialized expertise** → Subagents

Understanding when and why to use each layer transforms Claude Code from a chatbot into an intelligent development infrastructure. Start simple, add layers as justified by real pain points, and maintain ruthless focus on value over complexity.

The best configuration is the simplest one that solves your actual problems.
