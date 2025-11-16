# How to Configure CLAUDE.md Effectively

## Problem

Claude Code doesn't understand your project's specific context, leading to:

- Repeated explanations of tech stack and conventions
- Incorrect assumptions about directory structure
- Suggestions that violate project standards
- Wasted context tokens on basic clarifications
- Inconsistent code across sessions

**Example frustrations:**
- "Why does Claude keep suggesting the wrong test framework?"
- "I have to explain our data schema every time"
- "Claude doesn't know which commands to run"

## Solution

Create a CLAUDE.md file that provides essential project context automatically. This file loads once at session start, ensuring Claude understands your project without repeated explanations.

## Prerequisites

- Claude Code installed
- A project directory (new or existing)
- 30-60 minutes to document your project

## Step-by-Step Guide

### Step 1: Choose Location

CLAUDE.md can exist at three levels (hierarchical, all load together):

**Project-level (recommended for teams):**
```bash
# Root of your project
touch CLAUDE.md
# OR
mkdir -p .claude
touch .claude/CLAUDE.md
```

**User-level (personal preferences):**
```bash
# Applies to all your projects
touch ~/.claude/CLAUDE.md
```

**Language/module-level (advanced):**
```bash
# Python-specific context
touch python/CLAUDE.md

# R-specific context
touch r/CLAUDE.md
```

**Hierarchy:** Enterprise → User → Project → Module (all merge)

### Step 2: Start with Minimal Template

**Don't overwhelm yourself.** Start with essentials only:

```markdown
# [Project Name]

## Tech Stack
- [Primary language + version]
- [Key frameworks/libraries]
- [Database/data format]

## Key Commands
- `[command]` - [what it does]
- `[command]` - [what it does]

## Directory Structure
```
[simplified tree showing main directories]
```

## Critical Conventions
- [Convention 1]
- [Convention 2]
- [Convention 3]

## Protected Areas
- ❌ [What not to modify]
- ❌ [What not to modify]
```

**That's it for version 1.** Add more as you identify needs.

### Step 3: Complete Working Example - Python Data Science Project

```markdown
# Analytics Pipeline Project

## Tech Stack
- Python 3.11+
- pandas, numpy for data manipulation
- scikit-learn for ML
- pytest for testing
- black for formatting

## Environment Setup
Activate: `source venv/bin/activate` or `micromamba activate analytics`
Dependencies: `pip install -r requirements.txt`

## Key Commands
- `pytest tests/` - Run test suite
- `black src/` - Format code
- `python -m src.pipeline.run` - Execute pipeline
- `jupyter lab` - Start notebook server

## Directory Structure
```
src/
├── data/           # Data loading and validation
├── processing/     # Data transformation
├── models/         # ML models
└── utils/          # Shared utilities
data/
├── raw/           # Original, immutable data
└── processed/     # Cleaned data (Parquet format)
tests/             # Mirror src/ structure
```

## Code Conventions
- Type hints required for all functions
- Google-style docstrings
- Use pathlib, never os.path
- Prefer pandas over raw loops
- Set random seeds to 42

## Data Patterns
- Load from data/processed/ using pandas.read_parquet()
- Save to data/processed/ or models/ depending on artifact type
- Validate with src/utils/validators.py

## Testing Strategy
- Unit tests mirror src/ structure in tests/
- Use fixtures from tests/conftest.py
- Minimum 80% coverage required

## Protected Areas
- ❌ Never modify data/raw/ files
- ❌ Don't use CSV for large data (slow, loses types)
- ❌ No hardcoded credentials (use environment variables)

## Data Schema
Main dataset: customers.parquet
- customer_id: int64, unique identifier
- created_at: datetime64[ns], UTC timezone
- status: string, one of ['active', 'inactive', 'churned']
- lifetime_value: float64, range [0, 100000]
```

### Step 4: Example for Multi-Language Project

```markdown
# Multi-Language Analytics Platform

## Language Responsibilities
- Python: Data engineering, ML models, API services
- R: Statistical analysis, reporting, Shiny dashboards
- Shared: Data interchange via Parquet format

## Data Flow
Python ETL → data/processed/*.parquet → R analysis → reports/

## Common Commands
- `make test-all`: Run tests across all languages
- `make setup`: Initialize all environments
- `docker-compose up`: Start development stack

## Cross-Language Conventions
- Use pathlib (Python) / here package (R) for paths
- UTC timezone for all timestamps
- Snake_case for variable names in both languages
- Maximum line length: 100 characters

## Architecture
See architecture/decisions/ for ADRs

Language-specific details:
- Python context: python/CLAUDE.md
- R context: r/CLAUDE.md

## Protected Areas
- ❌ Never modify raw data files in data/raw/
- ❌ Don't duplicate code between languages (use shared/)
- ❌ No hardcoded credentials (use environment variables)
```

Then create language-specific files:

**python/CLAUDE.md:**
```markdown
# Python Module Context

## Environment
- Python 3.11+ required
- Activate: `source venv/bin/activate`
- Dependencies: `pip install -r requirements.txt`

## Key Libraries
- pandas, numpy for data manipulation
- scikit-learn for ML
- pytest for testing
- black for formatting

## Code Conventions
- Type hints required for all functions
- Google-style docstrings
- Use pathlib, never os.path
- Prefer pandas over raw loops
- Set random seeds to 42

## Data Patterns
Input: Load from data/processed/ using pandas.read_parquet()
Output: Save to data/processed/ or models/
Validation: Use src/utils/validators.py for schema checks

## Testing Strategy
- Unit tests mirror src/ structure in tests/
- Use fixtures from tests/conftest.py
- Parametrize tests for multiple scenarios
- Minimum 80% coverage required
```

**r/CLAUDE.md:**
```markdown
# R Module Context

## Environment
- R 4.3+ required
- Activate: `source renv/activate.R`
- Dependencies: `renv::restore()`

## Key Packages
- tidyverse for data wrangling
- arrow for Parquet I/O
- ggplot2 for visualization
- testthat for testing

## Code Conventions
- Follow tidyverse style guide
- Use %>% pipe operator
- Prefer dplyr over base R
- Document with roxygen2
- Use snake_case consistently

## Data Patterns
Input: arrow::read_parquet("data/processed/file.parquet")
Output: Reports to reports/, models to models/
Validation: Use assertr package for data validation

## Testing Strategy
- Tests in tests/testthat/
- Descriptive test names: test_that("function handles missing values")
- Use fixtures from tests/fixtures/
```

### Step 5: What to Include (and Exclude)

**DO include:**
- ✓ Non-obvious project-specific information
- ✓ Critical commands (build, test, deploy)
- ✓ Data schemas and expected formats
- ✓ Coding standards specific to your project
- ✓ Directory structure and purpose
- ✓ Protected areas (what not to modify)
- ✓ Methodological preferences (statistical methods, etc.)

**DON'T include:**
- ✗ General programming knowledge
- ✗ Language syntax basics
- ✗ Framework documentation (link instead)
- ✗ Detailed implementation code
- ✗ Everything from your wiki

**Golden rule:** If you explain it in every new session, it belongs in CLAUDE.md.

### Step 6: Keep It Concise

Target: Under 100 lines for single-language projects

**Why?** Every line consumes context tokens in every interaction.

**How to stay lean:**

1. **Use references for details:**
```markdown
## Architecture Decisions
See architecture/decisions/ for ADRs

## API Documentation
Full spec: docs/api-spec.yaml
```

2. **Modularize with imports:**
```markdown
## Personal Preferences
Personal coding style preferences
```

3. **Link to external docs:**
```markdown
## Framework Patterns
Standard patterns: docs/patterns/
Database: docs/patterns/database-connection.md
Testing: docs/patterns/test-structure.md
```

### Step 7: Test Your Configuration

1. **Start fresh session:**
   ```bash
   claude
   ```

2. **Ask project-specific questions without explanation:**
   - "Run the tests"
   - "Format the code in src/"
   - "What's our data schema for customers?"

3. **Check if Claude:**
   - Uses correct commands
   - Follows coding conventions
   - Understands directory structure
   - Respects protected areas

4. **Refine based on gaps**

## Advanced Patterns

### Import System

For large projects, split context into files:

**Root CLAUDE.md:**
```markdown
# Main Project

## Core Context
[Essential info here]

## Additional Context
Architecture decisions
Personal preferences
```

**Note:** Reference with file path only (no @ prefix needed in CLAUDE.md imports).

### Dynamic Updates During Session

Claude Code's `/memory` command adds information during sessions:

```
User: /memory # Use median imputation for missing numerical values
```

Items added with `#` prefix automatically update CLAUDE.md.

### Version Control

**Commit CLAUDE.md to git** for team consistency:

```bash
git add CLAUDE.md
git commit -m "Add project context for Claude Code"
```

**Use .gitignore for personal settings:**
```gitignore
# .gitignore
CLAUDE.local.md
.claude/settings.local.json
```

## Complete Real-World Example - R Package Development

```markdown
# [Package Name] R Package

## Package Structure
- R/ - Function definitions (exported and internal)
- man/ - Auto-generated documentation (from roxygen2)
- tests/testthat/ - Unit tests
- vignettes/ - Long-form documentation
- data/ - Example datasets
- data-raw/ - Scripts creating data/ contents

## Development Commands
- `devtools::load_all()` - Load package for testing
- `devtools::document()` - Generate documentation from roxygen
- `devtools::test()` - Run test suite
- `devtools::check()` - Full package validation
- `devtools::install()` - Install locally

## Code Standards
- Follow tidyverse style guide
- Use roxygen2 for documentation
- Export functions explicitly with @export
- Document all parameters with @param
- Include examples in @examples

## Testing Philosophy
- Always run tests before proposing changes
- Never remove tests to make them pass
- Ask for human input when tests fail unexpectedly
- Test user-facing functions thoroughly
- Use fixtures for complex test data

## Protected Areas
- ❌ Never modify man/ directly (generated from roxygen)
- ❌ Don't change package dependencies without discussion
- ❌ Don't alter exported function signatures (breaking changes)

## Key Dependencies
Read documentation for:
- ?dplyr::filter - Data filtering
- ?ggplot2::ggplot - Plotting fundamentals
- ?tidyr::pivot_longer - Data reshaping
```

## Troubleshooting

### Claude Doesn't Seem to Read CLAUDE.md

**Symptoms:** Claude asks questions answered in CLAUDE.md

**Solutions:**
1. Verify file location: Project root or `.claude/CLAUDE.md`
2. Check file name: Must be exactly `CLAUDE.md` (all caps)
3. Restart Claude Code session
4. Verify file isn't empty or corrupted
5. Check for YAML frontmatter issues (if used)

### CLAUDE.md Too Long, Session Slow

**Symptoms:** Slow responses, hitting context limits quickly

**Solutions:**
1. Reduce to essentials (target <100 lines)
2. Move details to separate docs and reference them
3. Use file path references instead of inline content
4. Split hierarchical files (python/CLAUDE.md, r/CLAUDE.md)
5. Remove general knowledge (only project-specific)

### Context Not Loading from Subdirectory

**Symptoms:** Python-specific context not active in python/ directory

**Solutions:**
1. Verify file exists: `python/CLAUDE.md`
2. Check you're actually in that directory
3. Hierarchical loading requires all files (root + subdirectory)
4. Reference parent context if needed

### Team Members Getting Different Behavior

**Cause:** Personal ~/.claude/CLAUDE.md overriding project settings

**Solutions:**
1. Document which settings are personal vs. team
2. Use CLAUDE.local.md for personal overrides (gitignored)
3. Keep team settings in project CLAUDE.md (version controlled)
4. Clearly separate concerns in documentation

## Best Practices

### DO

- ✓ Start minimal, expand based on repeated clarifications
- ✓ Version control project CLAUDE.md
- ✓ Use hierarchical files for multi-language projects
- ✓ Document data schemas explicitly
- ✓ Include methodological preferences
- ✓ Update when adding major features
- ✓ Keep under 100 lines when possible

### DON'T

- ✗ Include general programming knowledge
- ✗ Duplicate framework documentation
- ✗ Create one giant file for everything
- ✗ Forget to test after major updates
- ✗ Mix personal preferences with team standards

## Maintenance Strategy

**Monthly review:**
1. What questions are you still explaining repeatedly?
2. What new conventions emerged?
3. What's no longer relevant?
4. Is the file getting too long?

**Update triggers:**
- New team member onboarding difficulties
- Repeated clarifications in sessions
- Major technology changes
- New coding standards adopted

## Next Steps

1. **Create your first CLAUDE.md** using the minimal template
2. **Test it** in a fresh session
3. **Iterate** based on gaps you notice
4. **Share with team** (if applicable)
5. **Maintain** as project evolves

## Related Guides

- `docs/manuals/claude/how-to/create-slash-command.md` - Automate repetitive workflows
- `docs/manuals/claude/how-to/create-specialized-agent.md` - Complex stateful tasks
- `docs/manuals/claude/reference/claude-md-schema.md` - Complete CLAUDE.md reference

## Additional Resources

- Official Anthropic CLAUDE.md documentation
- Example CLAUDE.md files: github.com/hesreallyhim/awesome-claude-code
- Template marketplace: aitmpl.com
