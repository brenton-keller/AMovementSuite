# How to Create Your First Slash Command

## Problem

You find yourself repeatedly asking Claude Code to perform the same task with similar instructions. Each time requires re-explaining the context, steps, and expected output. This wastes time and context tokens.

**Examples of repetitive tasks:**
- Running test suites across different languages
- Generating data quality reports
- Formatting code with specific tools
- Creating standardized documentation

## Solution

Create a reusable slash command that captures the workflow as a template. Commands live in `.claude/commands/` (project-wide) or `~/.claude/commands/` (personal) and can accept dynamic arguments.

## Prerequisites

- Claude Code installed and working in your project
- Basic understanding of Markdown
- Familiarity with YAML frontmatter (optional but recommended)

## Step-by-Step Guide

### Step 1: Identify Your Workflow

Before creating a command, document what you're automating:

1. What problem does this solve?
2. What are the steps involved?
3. What inputs vary between runs?
4. What tools are needed (Bash, Read, Write, Edit)?

**Example:** "I need to run tests for Python, R, and JavaScript projects automatically."

### Step 2: Create the Command File

Commands are Markdown files with optional YAML frontmatter. Create the directory and file:

```bash
# For project-specific command
mkdir -p .claude/commands
touch .claude/commands/test.md

# For personal command (all projects)
mkdir -p ~/.claude/commands
touch ~/.claude/commands/test.md
```

### Step 3: Write Your Command

**Basic command structure:**

```markdown
---
description: Brief description shown in /help
allowed-tools: Bash, Read, Write
argument-hint: [optional-argument-description]
---

# Command Title

Brief explanation of what this command does.

## Instructions

1. Step-by-step instructions for Claude to follow
2. Use $ARGUMENTS for user-provided input
3. Be specific about expected behavior

## Examples

Show example usage:
- `/commandname` - Default behavior
- `/commandname arg1 arg2` - With arguments
```

**Complete working example - Test Command:**

```markdown
---
description: Run comprehensive test suite for current project
allowed-tools: Bash, Read
argument-hint: [test-path] [options]
---

# Run Tests

Execute tests appropriate for current project type with coverage reporting.

## Auto-Detection Strategy

1. **Detect Project Type**
   - Python: Check for pytest.ini, tox.ini, or tests/ with .py files
   - R: Check for testthat/ directory or tests/testthat/
   - JavaScript: Check for jest.config.js or test/ with .js/.ts files

2. **Execute Appropriate Test Runner**

   **Python:**
   ```bash
   pytest $ARGUMENTS --cov --cov-report=term-missing --cov-report=html
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
   - Highlight failures with context
   - Show coverage statistics
   - Suggest fixes for common failures

## Usage Examples

- `/test` - Run all tests
- `/test tests/test_data_loader.py` - Run specific test file
- `/test -k "test_missing_values"` - Run tests matching pattern
```

### Step 4: Use Positional Arguments (Advanced)

For structured inputs, use positional arguments `$1`, `$2`, etc.

**Example - Data Quality Report:**

```markdown
---
description: Generate data quality report with customizable checks
argument-hint: [dataset-path] [output-format] [quality-threshold]
---

# Data Quality Report

Generate comprehensive data quality assessment for: $1

## Configuration

- Dataset: $1
- Output format: $2 (html, pdf, or markdown)
- Quality threshold: $3% (warn if below)

## Quality Checks

1. **Completeness**
   - Missing value percentage by column
   - Record count validation

2. **Validity**
   - Data type conformance
   - Range validation (min/max checks)
   - Format validation (dates, emails, etc.)

3. **Uniqueness**
   - Duplicate detection
   - Primary key validation

4. **Consistency**
   - Cross-column validation rules
   - Referential integrity checks

## Output

Generate $2 report in reports/quality/ with:
- Executive summary
- Per-column quality metrics
- Failed validation details
- Recommended actions

## Quality Score

Calculate overall score (0-100) and warn if below $3%

## Usage

`/data-quality-report data/raw/customers.csv html 95`
```

### Step 5: Organize with Namespaces

For multiple commands, organize them in subdirectories:

```
.claude/commands/
├── data/
│   ├── validate.md
│   ├── clean.md
│   └── profile.md
├── ml/
│   ├── train.md
│   └── evaluate.md
└── shared/
    ├── test.md
    └── format.md
```

**Note:** Subdirectories create logical grouping visible in `/help` but don't change command names.

### Step 6: Test Your Command

1. Save the file
2. Run `/help` to verify Claude sees your command
3. Execute the command: `/test` or `/commandname args`
4. Verify output matches expectations
5. Refine based on results

## Complete Working Example - Format Command

This command works across Python, R, and JavaScript:

```markdown
---
description: Format code using appropriate formatter for detected language
allowed-tools: Bash, Read
argument-hint: [file-or-directory]
---

# Format Code

Automatically detect language and apply appropriate code formatter.

## Detection and Formatting

**Python:**
- Detect: .py files or pyproject.toml
- Format: `black $ARGUMENTS`
- Alternative: `ruff format $ARGUMENTS`

**R:**
- Detect: .R files or .Rproj
- Format: `Rscript -e "styler::style_file('$ARGUMENTS')"`

**JavaScript/TypeScript:**
- Detect: .js/.ts files or package.json
- Format: `npx prettier --write $ARGUMENTS`

## Workflow

1. Read file/directory path: $ARGUMENTS
2. Detect language based on file extensions
3. Run appropriate formatter
4. Report changes made
5. Warn if formatter not installed

## Usage

- `/format src/` - Format entire directory
- `/format main.py` - Format single file
- `/format .` - Format current directory

## Error Handling

If formatter not found:
- Python: Suggest `pip install black` or `pip install ruff`
- R: Suggest `install.packages("styler")`
- JS: Suggest `npm install -D prettier`
```

## Troubleshooting

### Command Not Showing in /help

**Cause:** File not in correct location or invalid YAML frontmatter

**Solutions:**
1. Verify file is in `.claude/commands/` or `~/.claude/commands/`
2. Check YAML frontmatter syntax (3 dashes before and after)
3. Restart Claude Code session
4. Run `/help` to refresh command list

### Arguments Not Being Passed

**Cause:** Using wrong argument syntax

**Solutions:**
1. Use `$ARGUMENTS` for all user input
2. Use `$1`, `$2` for positional arguments
3. Reference arguments correctly: `$1` not `{$1}` or `${1}`
4. Quote arguments if they contain spaces: `"$ARGUMENTS"`

### Command Runs But Doesn't Work as Expected

**Cause:** Instructions unclear or tools not allowed

**Solutions:**
1. Add required tools to `allowed-tools` in frontmatter
2. Make instructions more explicit (less ambiguous)
3. Add error handling steps
4. Test with `/clear` to ensure clean context

### Bash Commands Fail

**Cause:** Shell differences or missing dependencies

**Solutions:**
1. Use POSIX-compliant commands when possible
2. Check if required tools installed (pytest, ruff, npm)
3. Use full paths: `/usr/bin/python` instead of `python`
4. Add installation checks to command

## Best Practices

### DO

- ✓ Start with simple commands, add complexity gradually
- ✓ Use descriptive names: `/generate-report` not `/gr`
- ✓ Include usage examples in the command
- ✓ Make commands language-agnostic when possible
- ✓ Add error handling and helpful messages
- ✓ Test commands in clean context

### DON'T

- ✗ Create commands for one-off tasks
- ✗ Hardcode file paths (use arguments)
- ✗ Forget to document expected arguments
- ✗ Make commands too complex (split into multiple)
- ✗ Assume tools are installed (detect and suggest)

## Next Steps

1. **Start small:** Create one command for your most repetitive task
2. **Iterate:** Refine based on actual usage
3. **Share:** Commit project commands to version control
4. **Expand:** Create command library for common workflows
5. **Advanced:** Explore composite commands that chain multiple operations

## Related Guides

- `docs/manuals/claude/how-to/setup-claude-md.md` - Configure project context
- `docs/manuals/claude/how-to/create-specialized-agent.md` - For complex, stateful workflows
- `docs/manuals/claude/reference/slash-commands.md` - Complete command reference

## Additional Resources

- Official Anthropic documentation on slash commands
- Community command library: github.com/hesreallyhim/awesome-claude-code
- Template marketplace: aitmpl.com
