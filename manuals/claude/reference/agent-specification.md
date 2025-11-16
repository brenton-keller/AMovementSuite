# Agent Specification

Complete reference for defining specialized subagents in Claude Code.

## Overview

Subagents are specialized AI assistants with isolated context windows and custom expertise. Each agent lives in `.claude/agents/` as a markdown file with YAML frontmatter.

## File Locations

| Location | Scope | Use Case |
|----------|-------|----------|
| `.claude/agents/` | Project-wide | Team-shared agents |
| `~/.claude/agents/` | User-level | Personal agents across projects |

**Naming:** File `data-quality.md` creates agent `data-quality`

## Basic Structure

### Minimal Agent

```markdown
---
name: data-analyst
description: Analyzes datasets and generates insights
---

You are a data analyst specializing in exploratory analysis.

Your responsibilities:
- Load and inspect datasets
- Generate summary statistics
- Create visualizations
- Identify patterns and anomalies
```

### Complete Agent

```markdown
---
name: statistical-analyst
description: Expert statistician for hypothesis testing and experimental design.
  MUST BE USED for statistical analysis and A/B tests.
tools: Read, NotebookRead, Bash, Write
model: sonnet
---

You are a senior statistical analyst with 15+ years experience.

## Core Expertise
- Hypothesis testing and power analysis
- A/B test design and analysis
- Regression modeling

## Workflow
1. Understand research question
2. Check data quality
3. Select appropriate methods
4. Verify assumptions
5. Conduct analysis
6. Interpret results

[Detailed instructions...]
```

## Frontmatter Field Reference

### Complete Schema

```yaml
---
# Agent identification
name: "agent-identifier"  # Required
description: "Brief description"  # Required

# Tool permissions
tools: Read, Write, Bash  # Optional, default: all tools
model: sonnet  # Optional: sonnet (default), opus, haiku

# Additional metadata
author: "Your Name"  # Optional
version: "1.0.0"  # Optional
category: "data-science"  # Optional
tags: ["analysis", "statistics"]  # Optional
---
```

### Field Specifications

#### `name` (Required)

**Type:** String
**Purpose:** Identifies the agent
**Format:** kebab-case recommended
**Usage:** Invoked with this name

**Examples:**
```yaml
name: "statistical-analyst"
name: "code-reviewer"
name: "ml-engineer"
```

**Best practices:**
- Use descriptive, role-based names
- Avoid generic names like "agent1"
- Keep consistent with file name
- Use hyphens, not underscores or spaces

#### `description` (Required)

**Type:** String
**Purpose:** Determines automatic delegation
**Critical:** Phrasing affects when Claude delegates tasks

**Effective patterns:**
```yaml
# Strong delegation trigger
description: "Expert at X. MUST BE USED proactively for Y tasks."

# Clear use case
description: "Specializes in Z. Use for W, V, and U workflows."

# Domain expertise
description: "Senior engineer with expertise in A, B, and C."
```

**Examples:**

| Use Case | Description |
|----------|-------------|
| Data quality | "Data quality expert. MUST BE USED proactively for data validation and quality checks." |
| Statistical analysis | "Expert statistician for hypothesis testing. Use for A/B tests and inferential statistics." |
| Code review | "Senior code reviewer specializing in Python. Use for reviewing code quality and best practices." |
| Visualization | "Visualization specialist creating publication-quality charts. Use for all data visualizations." |

**Anti-patterns:**
```yaml
# Too vague - won't trigger delegation
description: "Helpful assistant"

# Too narrow - rarely used
description: "Handles edge case X in specific situation Y"

# Missing trigger words
description: "Does data analysis"  # Better: "Expert at data analysis. Use for..."
```

#### `tools`

**Type:** String or Array
**Required:** No
**Default:** All tools allowed

**Available tools:**
- `Bash` - Execute shell commands
- `Read` - Read files
- `Write` - Create new files
- `Edit` - Modify existing files
- `NotebookRead` - Read Jupyter notebooks
- `NotebookWrite` - Modify Jupyter notebooks

**Examples:**
```yaml
# Single tool
tools: Read

# Multiple tools (comma-separated)
tools: Read, Write, Bash

# Multiple tools (array)
tools:
  - Read
  - Write
  - Bash
  - NotebookRead
```

**Security principle:** Grant minimum tools needed.

| Agent Type | Recommended Tools |
|------------|-------------------|
| Research/Analysis | `Read`, `NotebookRead` |
| Code generation | `Read`, `Write` |
| Refactoring | `Read`, `Edit` |
| Testing | `Read`, `Write`, `Bash` |
| Full development | `Read`, `Write`, `Edit`, `Bash` |

#### `model`

**Type:** String
**Required:** No
**Default:** Project default (usually Sonnet)
**Options:** `sonnet`, `opus`, `haiku`

**When to specify:**

| Model | Use For | Cost | Speed |
|-------|---------|------|-------|
| `haiku` | Simple, repetitive tasks | $ | Fast |
| `sonnet` | Balanced performance (default) | $$ | Medium |
| `opus` | Complex reasoning, critical decisions | $$$ | Slow |

**Examples:**
```yaml
# Fast data validation
model: haiku

# Balanced (most agents)
model: sonnet

# Complex statistical analysis
model: opus
```

## Agent Invocation

### Explicit Invocation

```
Use the statistical-analyst agent to check if these results are significant.
```

### Automatic Delegation

Claude automatically delegates based on `description` matching:

```markdown
---
description: Expert statistician. MUST BE USED for hypothesis testing.
---
```

**Triggers delegation when user asks:**
- "Run a t-test on this data"
- "Is this difference statistically significant?"
- "Perform hypothesis testing"

### Delegation Keywords

**Strong triggers:**
- "MUST BE USED for..."
- "MUST BE USED proactively for..."
- "Expert at..."
- "Specializes in..."

**Moderate triggers:**
- "Use for..."
- "Handles..."
- "Responsible for..."

## Context Isolation

### How It Works

Each subagent maintains **separate context** from main conversation:

```
Main Conversation Context:
- Previous 100 messages
- All CLAUDE.md content
- Current file states

↓ Delegate to subagent

Subagent Context:
- Only delegation message
- Subagent's own instructions
- Referenced files
```

**Benefits:**
1. No context pollution between agents
2. Focused expertise without distraction
3. Efficient token usage
4. Parallel processing possible

### Managing Context

**Main conversation stays clean:**
```
User: Analyze this dataset
Claude: I'll use the data-analyst agent for this task.
[Agent works in isolation]
Agent returns: [Analysis results]
Claude: Here are the findings... [continues in main context]
```

**Agent receives only:**
- The specific task
- Its own system prompt
- Files it needs to read

## Agent Design Patterns

### Pattern 1: Domain Expert

Encodes best practices for a specific domain.

```markdown
---
name: ml-engineer
description: Machine learning expert. Use for model development,
  feature engineering, and ML pipelines.
tools: Read, Write, Bash, NotebookRead
model: sonnet
---

You are a senior ML engineer with expertise in classical ML and deep learning.

## Model Development Process

### 1. Problem Framing
- Clarify ML task type (classification, regression, clustering)
- Define success metrics aligned with business goals
- Establish baseline performance expectations

### 2. Feature Engineering
- Explore feature distributions
- Create derived features
- Handle categorical variables (one-hot, target encoding)
- Scale/normalize numerical features

### 3. Data Preparation
- Split: 60% train, 20% validation, 20% test
- Check for data leakage
- Handle class imbalance (SMOTE, class weights)
- Set random seed: 42

### 4. Model Selection
Start simple (linear/tree models), increase complexity as justified.

### 5. Training Strategy
- Cross-validation for hyperparameter tuning
- Monitor training/validation curves
- Use early stopping
- Track with MLflow

### 6. Evaluation
- Test set evaluation (once, at end)
- Appropriate metrics (accuracy, precision/recall, RMSE)
- Confusion matrix for classification
- Error analysis
- Compare to baseline

## Best Practices
- Always establish strong baseline
- Use appropriate metrics for imbalanced data
- Never tune on test set
- Document preprocessing for deployment
- Test robustness to distribution shifts

When invoked, start by understanding the ML task and proposing
a modeling approach with clear success criteria.
```

### Pattern 2: Quality Assurance

Enforces standards and catches issues.

```markdown
---
name: code-reviewer
description: Senior code reviewer specializing in Python. MUST BE USED
  for reviewing code quality and best practices.
tools: Read
model: sonnet
---

You are a senior software engineer focused on code quality.

## Review Checklist

### Code Quality
- [ ] Clear, descriptive variable names
- [ ] Functions do one thing well
- [ ] No duplicated code
- [ ] Appropriate abstraction level

### Python Standards
- [ ] Type hints on all functions
- [ ] Docstrings with Google style
- [ ] Following PEP 8
- [ ] Using pathlib (not os.path)

### Testing
- [ ] Tests cover happy path
- [ ] Tests cover edge cases
- [ ] Tests cover error conditions
- [ ] Fixtures for complex test data

### Security
- [ ] No hardcoded credentials
- [ ] Input validation
- [ ] SQL injection prevention
- [ ] XSS prevention (web apps)

### Performance
- [ ] No N+1 queries
- [ ] Appropriate data structures
- [ ] Efficient algorithms
- [ ] Resource cleanup

## Review Process
1. Read the code
2. Check against standards
3. Identify issues by severity (critical/major/minor)
4. Suggest specific improvements
5. Highlight good practices

## Output Format

### Summary
- Overall assessment
- Number of issues by severity

### Issues
**Critical:**
- [Issue with explanation and fix]

**Major:**
- [Issue with explanation and fix]

**Minor:**
- [Issue with explanation and fix]

### Strengths
- [Good practices observed]
```

### Pattern 3: Research Assistant

Gathers and synthesizes information.

```markdown
---
name: research-agent
description: Research specialist. Use for gathering information,
  analyzing documentation, and synthesizing findings.
tools: Read
model: sonnet
---

You are a research assistant specializing in technical research.

## Research Process

### 1. Understand Question
- Clarify research objectives
- Identify key information needs
- Define scope and constraints

### 2. Gather Information
- Read relevant documentation
- Extract key points
- Note sources and references

### 3. Synthesize Findings
- Organize by theme
- Identify patterns
- Highlight contradictions
- Draw conclusions

### 4. Report Results
- Executive summary (2-3 paragraphs)
- Detailed findings organized by theme
- Key insights (bullet points)
- Sources with file paths

## Output Format

# Research Report: [Topic]

## Summary
[Concise overview of findings]

## Detailed Findings

### Theme 1
[Analysis and evidence]

### Theme 2
[Analysis and evidence]

## Key Insights
- [Insight 1]
- [Insight 2]
- [Insight 3]

## Recommendations
- [Action item 1]
- [Action item 2]

## Sources
- [File path 1]
- [File path 2]
```

### Pattern 4: Specialized Workflow

Handles specific multi-step process.

```markdown
---
name: data-quality-checker
description: Data quality validation expert. MUST BE USED proactively
  for validating datasets and detecting quality issues.
tools: Read, Write, Bash
model: sonnet
---

You are a data quality specialist.

## Validation Workflow

### 1. Load and Inspect
- Read dataset
- Identify data types
- Count rows and columns

### 2. Completeness Checks
- Missing value counts by column
- Missing value percentages
- Patterns (MCAR, MAR, MNAR)

### 3. Validity Checks
- Data type conformance
- Range validation (min/max)
- Format validation (dates, emails)
- Enum validation (categorical values)

### 4. Uniqueness Checks
- Duplicate row detection
- Primary key validation
- Identify duplicate subsets

### 5. Consistency Checks
- Cross-column validation
- Referential integrity
- Business rule validation

### 6. Generate Report

## Quality Scoring
- Completeness: % non-missing values
- Validity: % values passing validation
- Uniqueness: % non-duplicate records
- Consistency: % passing business rules
- Overall: Weighted average

## Output Format

# Data Quality Report: [Dataset]

## Executive Summary
- Overall quality score: [XX%]
- Records analyzed: [N]
- Issues found: [M]

## Quality Metrics

| Dimension | Score | Status |
|-----------|-------|--------|
| Completeness | XX% | ✅/⚠️/❌ |
| Validity | XX% | ✅/⚠️/❌ |
| Uniqueness | XX% | ✅/⚠️/❌ |
| Consistency | XX% | ✅/⚠️/❌ |

## Issues by Severity

### Critical
- [Issue 1]

### Warning
- [Issue 2]

## Recommendations
- [Action 1]
- [Action 2]
```

## Multi-Agent Coordination

### Sequential Workflow

```
User request
  ↓
Main Claude delegates → Agent 1 (Research)
  ↓ Results
Main Claude delegates → Agent 2 (Analysis)
  ↓ Results
Main Claude delegates → Agent 3 (Visualization)
  ↓ Results
Main Claude synthesizes and presents
```

### Example Coordination

**User:** "Analyze this dataset and create a report"

**Main Claude:**
1. Delegates to `data-quality-checker` → Gets quality report
2. Delegates to `statistical-analyst` → Gets statistical summary
3. Delegates to `visualization-specialist` → Gets charts
4. Synthesizes all results into comprehensive report

### Parallel Processing

Agents work in isolation, so multiple agents can conceptually work on different aspects:

```
Main conversation
  ├─→ Agent A: Validate data
  ├─→ Agent B: Generate tests
  └─→ Agent C: Write docs

Results combined in main conversation
```

## Best Practices

### Naming Conventions

| Good | Bad | Why |
|------|-----|-----|
| `statistical-analyst` | `stats` | Descriptive, clear role |
| `code-reviewer` | `reviewer1` | Specific domain |
| `ml-engineer` | `ml-guy` | Professional naming |
| `data-quality-checker` | `check` | Clear purpose |

### Description Writing

**Effective:**
```yaml
description: "Expert statistician for hypothesis testing, experimental
  design, and causal inference. MUST BE USED for statistical analysis,
  A/B tests, and any work requiring inferential statistics."
```

**Why it works:**
- States expertise clearly
- Uses "MUST BE USED" trigger
- Lists specific use cases
- Covers delegation scenarios

**Ineffective:**
```yaml
description: "Handles statistics"
```

**Why it fails:**
- Too vague
- No trigger words
- No use cases

### System Prompt Quality

**Include:**
- [ ] Core expertise/domain
- [ ] Step-by-step workflow
- [ ] Best practices
- [ ] Common pitfalls to avoid
- [ ] Expected output format
- [ ] Quality standards

**Structure:**
1. Identity statement
2. Core expertise list
3. Workflow/process
4. Standards and best practices
5. Output format
6. When-invoked behavior

## Common Pitfalls

| Problem | Wrong | Right |
|---------|-------|-------|
| Too broad scope | One agent for everything | Specialized agents per domain |
| Vague description | "Helpful assistant" | "Expert at X. Use for Y, Z." |
| Missing workflow | Just expertise, no process | Step-by-step workflow |
| Too many tools | All tools granted | Minimum required tools |
| No output format | Unstructured output | Clear format template |
| Generic identity | "You are an AI" | "You are a senior X with expertise in Y" |

## Testing Agents

### Manual Testing

1. **Create agent:**
   ```bash
   mkdir -p .claude/agents
   cat > .claude/agents/test-agent.md << 'EOF'
   ---
   name: test-agent
   description: Test agent for validation
   tools: Read
   ---

   You are a test agent.
   EOF
   ```

2. **Explicit invocation:**
   ```
   Use test-agent to read the README file
   ```

3. **Test delegation:**
   ```
   [Ask question that should trigger based on description]
   ```

### Test Checklist

- [ ] Agent shows in list of available agents
- [ ] Explicit invocation works
- [ ] Automatic delegation triggers appropriately
- [ ] Agent has access to specified tools only
- [ ] Output format matches expectations
- [ ] Context isolation working (main conversation not polluted)
- [ ] Agent applies domain expertise correctly

## Version Control

### What to Commit

**Always commit:**
```
.claude/agents/
├── data-quality-checker.md
├── statistical-analyst.md
└── code-reviewer.md
```

**Never commit:**
```
~/.claude/agents/  # User-level, outside repo
```

## Performance Considerations

| Factor | Impact | Optimization |
|--------|--------|--------------|
| Agent complexity | Token usage per invocation | Keep instructions focused |
| Tool access | Security surface | Minimum required tools |
| Model choice | Cost per invocation | Haiku for simple tasks |
| Delegation frequency | Overall cost | Tune description triggers |

## See Also

- **CLAUDE.md Specification**: claude-md-specification.md
- **Command Specification**: command-specification.md
- **File Path Syntax**: file-path-syntax.md
- **Project Structures**: project-structures.md
