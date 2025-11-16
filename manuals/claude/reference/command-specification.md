# Slash Command Specification

Complete reference for creating and using slash commands in Claude Code.

## Overview

Slash commands extract repetitive workflows into reusable templates. They are markdown files with optional YAML frontmatter stored in `.claude/commands/` or `~/.claude/commands/`.

## File Locations

| Location | Scope | Use Case |
|----------|-------|----------|
| `.claude/commands/` | Project-wide | Team shared commands |
| `~/.claude/commands/` | User-level | Personal commands across all projects |
| `.claude/commands/namespace/` | Project namespace | Organized by domain |

**Naming:** File `test.md` creates command `/test`

## Basic Structure

### Minimal Command

```markdown
# Test Command

Run project tests and report results.

Execute appropriate test suite based on project type.
```

### With Frontmatter

```markdown
---
description: Run comprehensive test suite for current project
allowed-tools: Bash, Read
argument-hint: [test-path] [options]
---

# Run Tests

Execute tests appropriate for current project type with coverage reporting.

[Command implementation details...]
```

## YAML Frontmatter Reference

### Complete Frontmatter Schema

```yaml
---
# Required: None (all fields optional)

# Description shown in /help output
description: "Brief description of what this command does"

# Tools the command is allowed to use
allowed-tools: Bash, Read, Write, Edit

# Argument hint shown to user
argument-hint: "[required-arg] [optional-arg]"

# Model preference for this command
model: "sonnet"  # or "opus", "haiku"

# Auto-approve specific tools (use with caution)
auto-approve: Bash

# Additional metadata (informational)
author: "Your Name"
version: "1.0.0"
category: "testing"
tags: ["test", "quality"]
---
```

### Frontmatter Field Specifications

#### `description`

**Type:** String
**Required:** No
**Purpose:** Shows in `/help` command list
**Best practices:**
- Keep under 80 characters
- Focus on what, not how
- Use active voice

**Examples:**
```yaml
description: "Run comprehensive test suite for current project"
description: "Generate data quality report with customizable checks"
description: "Format code using project-specific style guide"
```

#### `allowed-tools`

**Type:** String or Array
**Required:** No
**Default:** All tools allowed
**Options:** `Bash`, `Read`, `Write`, `Edit`, `NotebookRead`, `NotebookWrite`

**Examples:**
```yaml
# Single tool
allowed-tools: Bash

# Multiple tools (comma-separated)
allowed-tools: Bash, Read, Write

# Multiple tools (array format)
allowed-tools:
  - Bash
  - Read
  - Write
```

**Tool Descriptions:**

| Tool | Purpose | Use Cases |
|------|---------|-----------|
| `Bash` | Execute shell commands | Running tests, building, deploying |
| `Read` | Read file contents | Analyzing code, checking configs |
| `Write` | Create new files | Generating code, creating reports |
| `Edit` | Modify existing files | Refactoring, updating configs |
| `NotebookRead` | Read Jupyter notebooks | Analyzing data science work |
| `NotebookWrite` | Modify Jupyter notebooks | Updating analyses |

#### `argument-hint`

**Type:** String
**Required:** No
**Purpose:** Shows expected arguments in command help
**Format:** Use `[required]` and `[optional]` brackets

**Examples:**
```yaml
argument-hint: "[dataset-path]"
argument-hint: "[test-path] [options]"
argument-hint: "[source] [destination] [--flags]"
```

#### `model`

**Type:** String
**Required:** No
**Default:** Project default model
**Options:** `sonnet`, `opus`, `haiku`

**When to specify:**
```yaml
# Use Haiku for fast, simple commands
model: "haiku"

# Use Sonnet for balanced performance (default)
model: "sonnet"

# Use Opus for complex reasoning
model: "opus"
```

#### `auto-approve`

**Type:** String or Array
**Required:** No
**Default:** None
**Warning:** ⚠️ Security risk - use sparingly

**Examples:**
```yaml
# Auto-approve single tool
auto-approve: Bash

# Auto-approve multiple tools
auto-approve: Bash, Read
```

**Security considerations:**
- Only auto-approve for trusted, well-tested commands
- Avoid auto-approve with `Write` or `Edit` on production systems
- Document why auto-approve is necessary
- Review command behavior regularly

## Argument System

### Basic Arguments

```markdown
# Command using $ARGUMENTS

Run analysis on: $ARGUMENTS

This will process the dataset and generate insights.
```

**Usage:** `/analyze data/customers.csv`
**Result:** `$ARGUMENTS` → `data/customers.csv`

### Positional Arguments

```markdown
# Command using $1, $2, $3

## Configuration
- Source: $1
- Destination: $2
- Format: $3

Process data from $1, transform it, and save to $2 in $3 format.
```

**Usage:** `/transform data/raw/input.csv data/processed/output.parquet parquet`
**Result:**
- `$1` → `data/raw/input.csv`
- `$2` → `data/processed/output.parquet`
- `$3` → `parquet`

### Default Values

```markdown
# Command with defaults

## Configuration
- Dataset: $1
- Output format: $2 (default: html if not specified)
- Quality threshold: $3 (default: 95 if not specified)

Generate $2 report for $1 with $3% quality threshold.
```

## Command Organization

### Flat Structure

```
.claude/commands/
├── test.md
├── format.md
├── lint.md
└── deploy.md
```

**Commands:** `/test`, `/format`, `/lint`, `/deploy`

### Namespace Structure

```
.claude/commands/
├── data/
│   ├── validate.md
│   ├── clean.md
│   └── profile.md
├── ml/
│   ├── train.md
│   ├── evaluate.md
│   └── deploy.md
└── shared/
    ├── test.md
    └── format.md
```

**Commands:**
- `/validate` (labeled `project:data` in `/help`)
- `/clean` (labeled `project:data`)
- `/train` (labeled `project:ml`)
- `/test` (labeled `project:shared`)

**Note:** Subdirectories organize but don't affect command names. They create logical grouping visible in `/help` output.

## Command Patterns

### Pattern 1: Context-Aware Commands

Detect project type and route accordingly.

```markdown
---
description: Run appropriate test suite for current project
allowed-tools: Bash, Read
---

# Run Tests

## Detection Strategy

1. **Detect Project Type**
   - Python: Check for pytest.ini, tox.ini, or tests/ with .py files
   - R: Check for testthat/ directory
   - JavaScript: Check for jest.config.js or test/ with .js/.ts files

2. **Execute Appropriate Runner**

   **Python:**
   ```bash
   pytest $ARGUMENTS --cov --cov-report=term-missing
   ```

   **R:**
   ```bash
   Rscript -e "devtools::test('$ARGUMENTS')"
   ```

   **JavaScript:**
   ```bash
   npm test -- $ARGUMENTS --coverage
   ```

3. **Report Results**
   - Display test summary
   - Highlight failures
   - Show coverage statistics
```

### Pattern 2: Parameterized Commands

Use arguments for configuration.

```markdown
---
description: Generate data quality report with customizable checks
argument-hint: [dataset-path] [output-format] [quality-threshold]
allowed-tools: Read, Write, Bash
---

# Data Quality Report

Generate comprehensive quality assessment for: $1

## Configuration
- Dataset: $1
- Output format: $2 (html, pdf, or markdown)
- Quality threshold: $3% (warn if below)

## Quality Checks

1. **Completeness**: Missing value percentage by column
2. **Validity**: Data type conformance, range validation
3. **Uniqueness**: Duplicate detection, primary key validation
4. **Consistency**: Cross-column validation rules

## Output

Generate $2 report in reports/quality/ with:
- Executive summary
- Per-column quality metrics
- Failed validation details
- Recommended actions

Calculate overall score (0-100) and warn if below $3%

## Usage Examples
- `/data-quality-report data/raw/customers.csv html 95`
- `/data-quality-report data/sales.parquet pdf 90`
```

### Pattern 3: Composite Commands

Chain multiple operations.

```markdown
---
description: Complete ML experiment workflow from data to report
allowed-tools: Bash, Read, Write, Edit
---

# ML Experiment Workflow

Execute complete pipeline: $ARGUMENTS

## Pipeline Stages

### 1. Data Preparation
- Load and validate data
- Handle missing values
- Feature engineering
- Train/validation/test split

### 2. Model Training
- Train with cross-validation
- Track experiments with MLflow
- Save trained models

### 3. Model Evaluation
- Test set performance
- Generate confusion matrix
- Calculate metrics (precision, recall, F1)
- Compare to baseline

### 4. Report Generation
- Compile results
- Create visualizations
- Document methodology
- Generate shareable report

## Error Handling

If any stage fails:
- Stop pipeline
- Report which stage failed
- Show error details
- Suggest fixes
- Provide recovery instructions

## Usage
`/ml-experiment customer-churn-prediction`
```

### Pattern 4: Cross-Language Commands

Work across different languages.

```markdown
---
description: Format code using appropriate language formatter
allowed-tools: Bash, Read
---

# Format Code

Apply consistent formatting to: $ARGUMENTS

## Language Detection

**Python (.py):**
```bash
black $ARGUMENTS
isort $ARGUMENTS
```

**R (.R, .Rmd):**
```bash
Rscript -e "styler::style_file('$ARGUMENTS')"
```

**JavaScript/TypeScript (.js, .ts):**
```bash
prettier --write $ARGUMENTS
```

**JSON (.json):**
```bash
jq '.' $ARGUMENTS > temp && mv temp $ARGUMENTS
```

## Verification
- Show formatting changes
- Report any syntax errors
- Confirm successful formatting
```

## Discovery and Help

### List Available Commands

```
/help
```

**Shows:**
- Command name
- Description (from frontmatter)
- Namespace (if in subdirectory)
- Argument hints

### Command Details

```
/help command-name
```

**Shows:**
- Full description
- Argument requirements
- Usage examples
- Allowed tools

## Best Practices

### Design Principles

1. **Around workflows, not technologies** - `/test` not `/pytest`
2. **Language-agnostic when possible** - Auto-detect and route
3. **Clear argument expectations** - Use `argument-hint`
4. **Comprehensive error handling** - Guide user on failures
5. **Informative output** - Summarize results, highlight issues

### Naming Conventions

| Good | Bad | Why |
|------|-----|-----|
| `/test` | `/run-tests` | Concise, clear |
| `/format` | `/code-formatter` | Action-oriented |
| `/validate` | `/data-validation-check` | Too verbose |
| `/deploy` | `/d` | Too cryptic |

### Documentation Standards

Every command should include:
- [ ] Clear description in frontmatter
- [ ] Usage examples
- [ ] Expected arguments documented
- [ ] Error handling strategy
- [ ] Output format described

### Security Checklist

- [ ] Avoid `auto-approve` unless necessary
- [ ] Validate arguments before use
- [ ] Don't hardcode credentials
- [ ] Use environment variables for secrets
- [ ] Limit `allowed-tools` to minimum needed
- [ ] Document security implications

## Common Pitfalls

| Problem | Wrong | Right |
|---------|-------|-------|
| Too broad tool access | `allowed-tools: All` | Specify only needed tools |
| Unsafe auto-approve | `auto-approve: Write, Edit, Bash` | Avoid or use sparingly |
| Hardcoded paths | `pytest tests/unit/` | Use `$ARGUMENTS` for flexibility |
| No error handling | Command fails silently | Check exit codes, report errors |
| Unclear arguments | `$1 $2 $3` without docs | Use `argument-hint` field |
| Missing description | No frontmatter | Add description for `/help` |

## Testing Commands

### Manual Testing

1. **Create test command:**
   ```bash
   mkdir -p .claude/commands/test
   echo "# Test" > .claude/commands/test/my-test.md
   ```

2. **Test in Claude:**
   ```
   /my-test arg1 arg2
   ```

3. **Verify behavior:**
   - Check arguments substituted correctly
   - Confirm tools execute as expected
   - Review output format
   - Test error cases

### Test Checklist

- [ ] Command shows in `/help`
- [ ] Description displays correctly
- [ ] Arguments substitute properly
- [ ] Tools execute without errors
- [ ] Error handling works
- [ ] Output is clear and actionable
- [ ] Works across different contexts
- [ ] Doesn't conflict with other commands

## Version Control

### What to Commit

**Always commit:**
```
.claude/commands/
├── data/
│   └── validate.md
└── shared/
    └── test.md
```

**Never commit:**
```
~/.claude/commands/  # User-level, outside repo
.claude/commands/personal/  # If using personal subdirectory
```

### .gitignore Recommended

```gitignore
# Personal commands (if using subdirectory)
.claude/commands/personal/
```

## Examples Library

### Example 1: Simple Test Command

```markdown
---
description: Run all tests with coverage
allowed-tools: Bash
---

# Test All

Run complete test suite with coverage reporting.

```bash
pytest --cov --cov-report=term-missing --cov-report=html
```

Coverage report available at: htmlcov/index.html
```

### Example 2: Data Validation

```markdown
---
description: Validate dataset schema and quality
argument-hint: [dataset-path]
allowed-tools: Read, Write, Bash
---

# Validate Data

Validate dataset: $1

## Checks

1. **Schema Validation**
   - Column names match expected
   - Data types correct
   - Required columns present

2. **Quality Checks**
   - No duplicate primary keys
   - Missing values within acceptable range
   - Value ranges as expected

3. **Output**
   - Generate validation report in reports/validation/
   - Exit code 0 if valid, 1 if issues found
```

### Example 3: Documentation Generator

```markdown
---
description: Generate API documentation from code
allowed-tools: Bash, Read, Write
model: sonnet
---

# Generate Docs

Generate documentation for: $ARGUMENTS

## Process

1. **Read code** in specified directory/file
2. **Extract** function signatures, docstrings, types
3. **Generate** markdown documentation
4. **Save** to docs/api/

## Output Format

```markdown
# Module: [name]

## Functions

### function_name(arg1: type, arg2: type) -> return_type

[Docstring content]

**Parameters:**
- arg1 (type): Description
- arg2 (type): Description

**Returns:**
- return_type: Description
```

Usage: `/generate-docs src/api/`
```

## Performance Tips

1. **Limit tool usage** - Only specify needed tools
2. **Use appropriate model** - Haiku for simple tasks
3. **Cache-friendly** - Avoid reading large files unnecessarily
4. **Clear instructions** - Reduces back-and-forth
5. **Progressive enhancement** - Start simple, add complexity as needed

## See Also

- **CLAUDE.md Specification**: claude-md-specification.md
- **Agent Specification**: agent-specification.md
- **File Path Syntax**: file-path-syntax.md
- **Project Structures**: project-structures.md
