# Claude Code for data science repositories: The complete implementation guide

**Claude Code transforms data science workflows through intelligent context management, specialized agents, and reusable commands.** This guide provides actionable patterns for implementing Claude Code across multi-language projects including Python, R, R Shiny, AutoHotkey v2, and APIs. Based on official Anthropic documentation and battle-tested community patterns, these practices enable 5-10x productivity improvements while maintaining code quality. Start with basic CLAUDE.md configuration, progress to custom slash commands and specialized subagents, then leverage advanced features like Model Context Protocol integration for seamless tool access. The key: treat documentation as living infrastructure that evolves with your project, not static setup that you write once and forget.

## Building blocks: The three-layer configuration system

Claude Code operates through a hierarchical configuration system that provides context at the right time and place. **The three layers—CLAUDE.md files for project memory, slash commands for reusable workflows, and subagents for specialized tasks—work together to create an intelligent development environment.**

CLAUDE.md files serve as automatic memory that loads when you start Claude Code. These markdown files live in your project root (`.claude/CLAUDE.md` or `CLAUDE.md`) and provide essential context: tech stack, directory structure, development commands, coding standards, and critical constraints. Unlike system prompts that apply globally, CLAUDE.md targets project-specific knowledge. The hierarchy matters: enterprise-level files set organizational policies, user-level files (`~/.claude/CLAUDE.md`) define personal preferences across all projects, and project-level files specify team conventions that get version-controlled with your code.

Effective CLAUDE.md files stay concise—under 100 lines when possible. Every line consumes context tokens with each interaction, so focus on non-obvious, project-specific information rather than general programming knowledge. Use the import system with `@path/to/file` syntax to modularize context. For example, `@docs/architecture/decisions.md` pulls in architectural decision records only when needed, while `@~/.claude/personal-preferences.md` incorporates your individual coding style. This progressive disclosure keeps base context lean while detailed information remains accessible.

For data science repositories, CLAUDE.md should document data schemas explicitly. Specify column names, data types, expected value ranges, and any domain-specific meanings. Include statistical methodology preferences: "use median imputation for numerical missing values," "apply Bonferroni correction for multiple comparisons," or "validate model assumptions before inference." Document commands for running tests (`pytest tests/`), starting notebooks (`jupyter lab`), and executing pipelines (`python -m src.pipeline.run`). This front-loaded context prevents repetitive clarifications and ensures Claude makes appropriate methodological choices.

**Slash commands extract repetitive workflows into reusable templates.** Store them in `.claude/commands/` for project-wide availability or `~/.claude/commands/` for personal use. Each command is a markdown file with optional YAML frontmatter specifying allowed tools, argument hints, descriptions, and model preferences. Commands use `$ARGUMENTS` as placeholders for dynamic inputs and support positional arguments like `$1`, `$2` for structured parameters. Organize commands in subdirectories—`.claude/commands/data/`, `.claude/commands/ml/`, `.claude/commands/analysis/`—to create namespaces that improve discoverability.

Commands shine for cross-language portability. A test command can detect context automatically: check for `pytest.ini` to run Python tests, `testthat/` directory for R tests, or `.ahk` extensions for AutoHotkey. This language-agnostic approach means one `/test` command serves your entire polyglot project. Similarly, a `/format` command can route to Black for Python, styler for R, or custom formatters based on file extensions. The key: design commands around workflows (test, lint, deploy) rather than specific technologies.

**Subagents are specialized AI assistants with isolated context windows and custom expertise.** Each subagent lives in `.claude/agents/` as a markdown file with frontmatter defining its name, description, allowed tools, and preferred model. The description field determines automatic delegation—Claude Code reads descriptions and routes tasks to appropriate subagents based on content matching. Phrasing matters: "MUST BE USED proactively for data quality validation" or "Expert at statistical hypothesis testing" helps Claude identify when to delegate.

Subagents maintain separate context, preventing main conversation pollution. A data-quality-checker subagent might receive only data files and validation rules, while a visualization-specialist subagent gets processed data and chart requirements. This isolation improves focus and efficiency. Subagents can use different models too—run compute-intensive subagents on Haiku for speed while reserving Sonnet for complex reasoning in the main conversation.

For data science workflows, create domain-specific subagents: data-engineer for pipeline construction, statistical-analyst for hypothesis testing, ml-engineer for model training, and report-generator for documentation. Each subagent encodes best practices for its domain. The statistical-analyst subagent knows to check assumptions before running parametric tests, calculate effect sizes alongside p-values, and document methodological choices. These encoded patterns ensure consistent quality without repeating instructions.

## Repository structure patterns for multi-language data science projects

**Effective repository organization makes Claude Code dramatically more productive by creating clear boundaries and explicit structure.** Multi-language data science projects require careful organization to prevent confusion between language ecosystems while maintaining cohesive workflows.

The recommended structure separates concerns by language while centralizing shared components:

```
project-root/
├── .claude/
│   ├── commands/          # Slash commands organized by function
│   │   ├── shared/        # Cross-language commands (setup, test-all)
│   │   ├── python/        # Python-specific commands
│   │   ├── r/            # R-specific commands
│   │   └── data/         # Data processing commands
│   ├── agents/           # Specialized subagents
│   │   ├── data-engineer.md
│   │   ├── ml-engineer.md
│   │   ├── statistical-analyst.md
│   │   └── code-reviewer.md
│   └── settings.json     # Project-level Claude settings
├── CLAUDE.md             # Root project documentation
├── .mcp.json            # MCP server configuration
├── docs/
│   ├── architecture/     # System design and ADRs
│   ├── patterns/        # Reusable code patterns
│   ├── languages/       # Language-specific setup guides
│   └── api/            # API reference documentation
├── data/
│   ├── raw/            # Original, immutable data
│   ├── processed/      # Cleaned, transformed data
│   └── schemas/        # Data schema definitions
├── python/             # Python codebase
│   ├── CLAUDE.md      # Python-specific context
│   ├── src/
│   ├── tests/
│   └── requirements.txt
├── r/                 # R codebase
│   ├── CLAUDE.md     # R-specific context
│   ├── R/
│   ├── tests/
│   └── renv.lock
├── notebooks/        # Jupyter/R notebooks
├── scripts/          # Standalone utility scripts
└── shared/          # Cross-language utilities
```

This structure provides clear navigation while enabling hierarchical context. When Claude Code navigates into `python/`, it automatically loads `python/CLAUDE.md` with Python-specific conventions. When working in `r/`, the R-specific context activates. The root `CLAUDE.md` provides overarching project context that applies everywhere.

**Hierarchical CLAUDE.md files create powerful contextual awareness.** The root-level file documents project-wide patterns:

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

## Architecture Decisions
See @docs/architecture/decisions/ for ADRs

## Protected Areas
- ❌ Never modify raw data files in data/raw/
- ❌ Don't duplicate code between languages (use shared/)
- ❌ No hardcoded credentials (use environment variables)
```

Language-specific CLAUDE.md files drill deeper:

```markdown
# Python Module Context

## Environment
- Python 3.11+ required
- Activate: `source venv/bin/activate` or `micromamba activate project`
- Dependencies: `pip install -r requirements.txt`

## Key Libraries
- pandas, numpy for data manipulation
- scikit-learn for ML
- pytest for testing
- black for formatting
- mypy for type checking

## Code Conventions
- Type hints required for all functions
- Google-style docstrings
- Use pathlib, never os.path
- Prefer pandas over raw loops
- Set random seeds to 42

## Data Patterns
Input: Load from data/processed/ using pandas.read_parquet()
Output: Save to data/processed/ or models/ depending on artifact type
Validation: Use @src/utils/validators.py for schema checks

## Testing Strategy
- Unit tests mirror src/ structure in tests/
- Use fixtures from tests/conftest.py
- Parametrize tests for multiple scenarios
- Minimum 80% coverage required
```

The R CLAUDE.md provides parallel structure with R-specific guidance:

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

This hierarchical approach means Claude Code automatically receives relevant context based on current working directory, reducing cognitive load and improving accuracy.

## Documentation structure that empowers AI assistance

**The /docs/ folder transforms from static reference material into active AI context when structured intentionally.** Effective documentation for Claude Code focuses on four categories: architectural decisions, reusable patterns, integration guides, and troubleshooting references.

Architecture Decision Records (ADRs) capture critical choices with context and consequences. Store these in `docs/architecture/decisions/` with sequential numbering:

```markdown
# ADR 003: Use Parquet for cross-language data interchange

## Status
Accepted

## Context
Need efficient data format that works seamlessly with both pandas (Python) 
and arrow/dplyr (R). CSV is slow and loses type information. Feather doesn't 
support complex nested types we need.

## Decision
Use Apache Parquet as standard format for all intermediate datasets between 
Python and R processing stages.

## Consequences
### Positive
- 10-20x faster than CSV for large datasets
- Preserves schema and types across languages
- Native support in both pandas and arrow
- Columnar format enables efficient querying
- Supports complex nested types

### Negative
- Binary format (not human-readable for debugging)
- Requires pyarrow (Python) and arrow (R) dependencies
- Slightly more complex than CSV for simple data

## Implementation
- All data/processed/ files use .parquet extension
- Python: pandas.read_parquet() / df.to_parquet()
- R: arrow::read_parquet() / arrow::write_parquet()
```

When Claude Code reads these ADRs (via `@docs/architecture/decisions/003-parquet.md` in CLAUDE.md), it understands not just what to do but why, enabling better decisions about similar issues.

**Pattern documentation captures reusable code templates and workflows.** These live in `docs/patterns/` and provide canonical implementations:

```markdown
# Pattern: Database Connection Management

## Overview
Standard pattern for database connections with pooling, retry logic, 
and proper cleanup.

## Python Implementation
```python
# src/db/connection.py
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool
import os

def get_db_connection():
    """
    Create database connection with connection pooling.
    
    Uses environment variables for configuration:
    - DB_HOST, DB_PORT, DB_NAME, DB_USER, DB_PASSWORD
    
    Returns connection from pool with automatic cleanup.
    """
    connection_string = (
        f"postgresql://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}"
        f"@{os.getenv('DB_HOST')}:{os.getenv('DB_PORT')}/{os.getenv('DB_NAME')}"
    )
    
    engine = create_engine(
        connection_string,
        poolclass=QueuePool,
        pool_size=5,
        max_overflow=10,
        pool_pre_ping=True  # Verify connections before use
    )
    
    return engine.connect()
```

## R Implementation
```r
# R/db_connection.R
library(DBI)
library(RPostgres)

get_db_connection <- function() {
  # Create connection with environment variables
  con <- dbConnect(
    RPostgres::Postgres(),
    host = Sys.getenv("DB_HOST"),
    port = Sys.getenv("DB_PORT"),
    dbname = Sys.getenv("DB_NAME"),
    user = Sys.getenv("DB_USER"),
    password = Sys.getenv("DB_PASSWORD")
  )
  
  # Verify connection
  stopifnot(dbIsValid(con))
  
  return(con)
}
```

## Usage
Always use in context manager / with block to ensure cleanup:

Python: `with get_db_connection() as conn: ...`
R: `con <- get_db_connection(); on.exit(dbDisconnect(con)); ...`

## When to Use
- All database access
- Multi-query transactions
- Long-running analysis scripts
- API endpoints querying database
```

Reference patterns in CLAUDE.md with anchor comments in code. Add `# PATTERN: database-connection` at implementation sites so Claude can find canonical examples when needed. This bidirectional linking—documentation referencing code, code referencing documentation—creates navigable knowledge graphs.

Integration guides in `docs/languages/` explain ecosystem-specific setup. For R projects, document RStudio integration, package development workflows, and renv usage. For Python, cover virtual environments, package managers (pip vs. poetry vs. conda), and type checking setup. These guides prevent Claude from making incorrect assumptions about your environment.

**Troubleshooting documentation in `docs/troubleshooting/` accelerates debugging.** Document common errors with solutions:

```markdown
# Common Data Pipeline Errors

## Error: "KeyError: 'customer_id'"
**Cause**: Missing expected column in input data
**Solution**: 
1. Check data schema matches expected: `df.info()`
2. Verify data source (file path, database query)
3. Update validation code if schema changed legitimately

## Error: "SettingWithCopyWarning" in pandas
**Cause**: Modifying view instead of copy
**Solution**: Always use `.copy()` when subsetting:
```python
# Wrong
filtered = df[df['age'] > 25]
filtered['new_col'] = value

# Right
filtered = df[df['age'] > 25].copy()
filtered['new_col'] = value
```
```

Reference these from CLAUDE.md so Claude checks troubleshooting docs when encountering errors.

## Creating portable slash commands that work across project types

**Portable commands abstract workflows from specific implementations, enabling reuse across languages and project types.** The key: design around intent rather than execution details, then handle language-specific implementation inside the command.

Context detection makes commands adaptive. A universal test command can detect project type and route accordingly:

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
   - AutoHotkey: Check for .ahk files and testing framework

2. **Execute Appropriate Test Runner**
   
   **Python:**
   ```bash
   pytest $ARGUMENTS --cov --cov-report=term-missing --cov-report=html
   ```
   
   **R:**
   ```bash
   Rscript -e "devtools::test('$ARGUMENTS')"
   # Or: Rscript -e "testthat::test_dir('tests/testthat/')"
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

This pattern scales to other cross-language workflows. A `/format` command detects language and applies appropriate formatter (Black for Python, styler for R, Prettier for JavaScript). A `/lint` command routes to pylint, lintr, or ESLint. A `/docs` command generates appropriate documentation (Sphinx for Python, roxygen2 for R, JSDoc for JavaScript).

**Parameterization through arguments makes commands flexible.** Use `$ARGUMENTS` for simple cases and positional arguments for structured inputs:

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

Composite commands chain multiple operations, useful for complex multi-step workflows:

```markdown
---
description: Complete ML experiment workflow from data to report
allowed-tools: Bash, Read, Write, Edit
---

# ML Experiment Workflow

Execute complete machine learning experiment pipeline: $ARGUMENTS

## Pipeline Stages

1. **Data Preparation** (via /prepare-data)
   - Load and validate data
   - Handle missing values
   - Apply feature engineering
   - Split train/validation/test

2. **Model Training** (via /train-model)
   - Train with cross-validation
   - Track experiments with MLflow
   - Save trained models

3. **Model Evaluation** (via /evaluate-model)
   - Test set performance
   - Generate confusion matrix
   - Calculate metrics (precision, recall, F1)
   - Compare to baseline

4. **Report Generation** (via /generate-ml-report)
   - Compile results
   - Create visualizations
   - Document methodology
   - Generate shareable report

## Error Handling
If any stage fails, stop pipeline and report:
- Stage that failed
- Error details
- Suggested fixes
- Recovery instructions

## Usage
`/ml-experiment customer-churn-prediction`
```

Organization through namespaces improves discoverability. Structure commands by domain:

```
.claude/commands/
├── data/
│   ├── validate.md       # /validate (project:data)
│   ├── clean.md         # /clean (project:data)
│   └── profile.md       # /profile (project:data)
├── ml/
│   ├── train.md         # /train (project:ml)
│   ├── evaluate.md      # /evaluate (project:ml)
│   └── deploy.md        # /deploy (project:ml)
├── analysis/
│   ├── eda.md           # /eda (project:analysis)
│   ├── hypothesis.md    # /hypothesis (project:analysis)
│   └── report.md        # /report (project:analysis)
└── shared/
    ├── test.md          # /test (project:shared)
    └── format.md        # /format (project:shared)
```

Subdirectories organize commands but don't affect command names—they create logical grouping visible in `/help` output as `(project:namespace)`.

## Implementing specialized agents for data science workflows

**Subagents encode domain expertise and best practices into autonomous specialists.** For data science, create agents that embody different roles: data engineer, statistical analyst, ML engineer, visualization specialist, and code reviewer. Each agent receives tailored system prompts that guide decision-making.

A statistical analyst subagent encodes rigorous methodology:

```markdown
---
name: statistical-analyst
description: Expert statistician for hypothesis testing, experimental design, 
  and causal inference. MUST BE USED for statistical analysis, A/B tests, 
  and any work requiring inferential statistics.
tools: Read, NotebookRead, Bash, Write
model: sonnet
---

You are a senior statistical analyst with 15+ years experience in applied 
statistics, experimental design, and causal inference.

## Core Expertise
- Hypothesis testing and power analysis
- A/B test design and analysis
- Regression modeling (linear, logistic, mixed-effects)
- Time series analysis
- Survival analysis
- Causal inference methods
- Bayesian statistics

## Analytical Workflow

### 1. Understand the Question
- Clarify research question and hypotheses
- Identify target population and sampling method
- Define primary and secondary outcomes
- Determine causal vs. associational goals

### 2. Check Data Quality
- Verify sample size and power
- Check for missing data patterns (MCAR, MAR, MNAR)
- Identify outliers and influential points
- Assess measurement quality and reliability

### 3. Select Methods
- Choose appropriate statistical tests for data type
- Consider parametric vs. non-parametric approaches
- Account for multiple comparisons if needed
- Plan sensitivity analyses

### 4. Verify Assumptions
Before any test, explicitly check:
- Normality (Q-Q plots, Shapiro-Wilk test)
- Homogeneity of variance (Levene's test)
- Independence (residual plots, Durbin-Watson)
- Linearity (for regression)

If assumptions violated, consider transformations or alternative methods.

### 5. Conduct Analysis
- Report effect sizes alongside p-values
- Calculate confidence intervals
- Perform sensitivity analyses
- Consider practical significance vs. statistical significance

### 6. Interpret Results
- Explain findings in plain language
- Discuss limitations and caveats
- Distinguish correlation from causation
- Provide actionable recommendations

## Best Practices
- Always start with exploratory visualization
- Calculate and report power for non-significant results
- Use multiple comparison corrections when appropriate (Bonferroni, FDR)
- Prefer estimation (confidence intervals) over hypothesis testing when possible
- Check residuals for model diagnostics
- Report both statistical and practical significance
- Document all methodological decisions
- Consider robustness checks and alternative specifications

## Code Standards
- Use scipy.stats for statistical tests in Python
- Use tidyverse + infer for R analysis
- Always set random seeds for reproducibility
- Include detailed comments explaining statistical choices
- Generate reproducible reports with R Markdown or Jupyter

## Common Pitfalls to Avoid
- Never p-hack or cherry-pick results
- Don't ignore assumption violations
- Don't confuse statistical significance with importance
- Don't run multiple tests without correction
- Don't interpret correlation as causation without appropriate design
- Don't use t-tests on non-normal data without justification

When invoked, immediately identify the statistical question, assess data quality, 
and propose an appropriate analytical approach before coding.
```

This encoding means every time the statistical-analyst subagent handles a task, it applies these principles automatically. No need to remind Claude about checking assumptions or reporting effect sizes—it's built into the subagent's identity.

A machine learning engineer subagent focuses on model development:

```markdown
---
name: ml-engineer
description: Machine learning expert specializing in model development, 
  feature engineering, and production ML pipelines. Use for training models, 
  hyperparameter tuning, and model evaluation.
tools: Read, Write, Bash, NotebookRead
model: sonnet
---

You are a senior ML engineer with expertise in classical ML and deep learning.

## Model Development Process

### 1. Problem Framing
- Clarify ML task type (classification, regression, clustering, etc.)
- Define success metrics aligned with business goals
- Establish baseline performance expectations
- Identify data requirements and constraints

### 2. Feature Engineering
- Explore feature distributions and relationships
- Create derived features based on domain knowledge
- Handle categorical variables appropriately (one-hot, target encoding, embeddings)
- Scale/normalize numerical features
- Address feature collinearity
- Document feature definitions

### 3. Data Preparation
- Split data: 60% train, 20% validation, 20% test (stratified if classification)
- Check for data leakage (temporal, target, feature)
- Handle class imbalance if present (SMOTE, class weights, stratified sampling)
- Create k-fold cross-validation splits
- Set random seed: 42

### 4. Model Selection
Choose models based on:
- Problem type and data size
- Interpretability requirements
- Training time constraints
- Deployment considerations

Start simple (linear/tree models), increase complexity as justified.

### 5. Training Strategy
- Use cross-validation for hyperparameter tuning
- Track experiments with MLflow or similar
- Monitor training/validation curves for overfitting
- Use early stopping when appropriate
- Save best models and configurations

### 6. Evaluation
- Test set evaluation (once, at the end)
- Calculate appropriate metrics (accuracy, precision/recall, RMSE, etc.)
- Generate confusion matrix for classification
- Analyze errors and failure modes
- Compare to baseline model
- Feature importance analysis

### 7. Model Interpretation
- SHAP values for feature impact
- Partial dependence plots
- Error analysis by subgroup
- Fairness assessment if applicable

## Best Practices
- Always establish strong baseline (simple model, domain rules)
- Use appropriate metrics for imbalanced data (F1, AUPRC, not just accuracy)
- Cross-validate hyperparameters, never tune on test set
- Document preprocessing steps for deployment
- Version control models and data
- Test model robustness to distribution shifts
- Monitor feature importance for unexpected patterns

## Code Standards
- Use scikit-learn pipelines for reproducibility
- Implement custom transformers for domain-specific logic
- Type hints for all functions
- Extensive docstrings with mathematical notation
- Separate training and inference code
- Include model serialization (joblib/pickle)

When invoked, start by understanding the ML task, exploring data characteristics, 
and proposing an modeling approach with clear success criteria.
```

A visualization specialist subagent ensures high-quality charts:

```markdown
---
name: visualization-specialist
description: Expert data visualization specialist creating publication-quality 
  charts, dashboards, and interactive visualizations. Use for creating or 
  improving any data visualizations.
tools: Read, Write, NotebookRead, Bash
model: sonnet
---

You are a senior data visualization expert specializing in effective communication 
through graphics.

## Visualization Selection
Choose chart types based on data and message:
- **Distributions**: Histograms, density plots, box plots, violin plots
- **Comparisons**: Bar charts, grouped bars, small multiples
- **Relationships**: Scatter plots, line charts, heatmaps
- **Compositions**: Stacked bars, pie charts (rarely), treemaps
- **Temporal**: Line charts, area charts, horizon plots
- **Geospatial**: Choropleth maps, symbol maps, cartograms

## Design Principles
1. **Clarity**: Remove chartjunk, maximize data-ink ratio
2. **Accessibility**: Colorblind-safe palettes, high contrast
3. **Accuracy**: Start axes at zero when appropriate, avoid misleading scales
4. **Context**: Descriptive titles, axis labels with units, legends when needed
5. **Aesthetics**: Professional appearance, consistent styling

## Implementation Standards

### Python (Matplotlib/Seaborn)
```python
import matplotlib.pyplot as plt
import seaborn as sns

# Set publication-quality defaults
sns.set_style("whitegrid")
sns.set_context("paper", font_scale=1.5)
plt.rcParams['figure.figsize'] = (10, 6)
plt.rcParams['figure.dpi'] = 300

# Use colorblind-friendly palettes
colors = sns.color_palette("colorblind")
```

### R (ggplot2)
```r
library(ggplot2)
library(viridis)  # Colorblind-safe palettes

theme_set(theme_minimal(base_size = 14))

# Consistent theme
custom_theme <- theme_minimal() +
  theme(
    plot.title = element_text(face = "bold"),
    axis.title = element_text(face = "bold"),
    legend.position = "bottom"
  )
```

## Accessibility Checklist
- Use colorblind-safe palettes (viridis, colorbrewer)
- Sufficient color contrast (4.5:1 ratio minimum)
- Avoid red/green combinations only
- Use shapes/patterns in addition to color
- Descriptive alt text for each visualization
- Font size ≥ 12pt for labels

## Best Practices
- Always include descriptive titles and axis labels
- Add units to axis labels (%, $, kg, etc.)
- Use thousands separators for large numbers (1,000,000)
- Format dates consistently (YYYY-MM-DD)
- Avoid 3D charts (rarely appropriate)
- Limit color palette to 5-7 distinct colors
- Use annotations to highlight key insights
- Save in vector format (SVG, PDF) when possible

When creating visualizations, first understand the message to communicate, 
then select appropriate chart type and design for clarity and impact.
```

**Subagent coordination enables complex workflows through delegation.** The main conversation can orchestrate multiple subagents for different aspects of a project. For a complete analysis pipeline:

1. Data-engineer subagent: Loads, validates, and preprocesses data
2. Statistical-analyst subagent: Performs hypothesis tests and statistical modeling
3. Ml-engineer subagent: Trains and evaluates predictive models
4. Visualization-specialist subagent: Creates compelling charts
5. Code-reviewer subagent: Reviews all code for quality and best practices

Each subagent works in isolation with focused context, returning results to the main conversation for integration. This prevents context pollution and allows specialized expertise at each stage.

## Advanced Claude Code features beyond the basics

**Extended thinking provides deeper reasoning for complex problems.** Trigger progressively more intensive reasoning with specific keywords: "think" for standard reasoning, "think hard" for deeper analysis, "think harder" for complex debugging, and "ultrathink" for architectural decisions requiring extensive deliberation. Each level allocates more thinking budget, visible in thinking blocks that show Claude's reasoning process. For data science, use extended thinking when debugging unexpected statistical results, designing complex experiments, or resolving methodological dilemmas.

**Hooks enable deterministic automation at Claude Code lifecycle events.** Configure hooks in `.claude/settings.json` to execute shell commands at specific points: `PreToolUse` before tool execution, `PostToolUse` after tools complete, `UserPromptSubmit` when prompts are submitted, `SessionStart`/`SessionEnd` for session boundaries, and `Notification` when awaiting input. Hooks receive event data via stdin as JSON and control flow through exit codes—exit 0 continues normally, exit 2 aborts the operation.

Practical hooks for data science include automatic code formatting, test execution verification, and logging. Format Python code automatically after edits:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.file_path' | { read file; if echo \"$file\" | grep -q '\\.py$'; then black \"$file\"; fi; }"
          }
        ]
      }
    ]
  }
}
```

Block dangerous commands proactively:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.command' | grep -qE 'rm -rf /|dd if=' && exit 2 || exit 0"
          }
        ]
      }
    ]
  }
}
```

Log all data processing operations for audit trails:

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "jq -r '.tool_input.command' >> ~/.claude/bash-audit.log"
          }
        ]
      }
    ]
  }
}
```

Keep hooks fast—they add latency to every operation. Test thoroughly before deploying to avoid breaking workflows.

**Model Context Protocol (MCP) connects Claude Code to external systems through standardized interfaces.** MCP is "USB-C for AI"—a universal protocol for tool integration. Configure MCP servers in `.mcp.json` for project-level availability or `~/.claude/mcp.json` for user-level access.

For data science, critical MCP integrations include database access (PostgreSQL, MySQL, SQLite servers), cloud storage (S3, Google Cloud Storage), data sources (Notion, Slack, Gmail for data collection), and web access (Puppeteer for web scraping). Configure a PostgreSQL MCP server:

```json
{
  "mcpServers": {
    "postgres": {
      "command": "npx",
      "args": [
        "-y",
        "@modelcontextprotocol/server-postgres",
        "postgresql://localhost/analytics"
      ],
      "env": {
        "PGUSER": "${DB_USER}",
        "PGPASSWORD": "${DB_PASSWORD}"
      }
    }
  }
}
```

Use environment variables for credentials—never hardcode secrets. Grant tool permissions explicitly using the MCP namespace: `mcp__postgres__query` grants query access, while blanket `mcp__postgres` approves all tools from that server.

**Plan mode separates research from execution.** Activate with Shift+Tab twice to enter planning mode where Claude analyzes and proposes approaches without making changes. Review the plan, provide feedback, then exit plan mode to execute. This prevents premature action when exploration is needed first. For data science, use plan mode when approaching unfamiliar datasets, debugging complex statistical issues, or designing experimental protocols.

**Compaction and checkpointing manage context limits.** Claude Code automatically checkpoints conversation state before significant changes. Press Escape twice or use `/rewind` to restore previous states. Manual compaction via `/compact [instructions]` summarizes conversation history when approaching context limits, preserving key information while freeing space. For data science sessions that accumulate extensive exploratory analysis, compact after completing distinct phases: "compact everything before feature engineering section" preserves recent context while clearing early exploration.

**Headless mode enables CI/CD integration and automation.** Run Claude Code non-interactively for pre-commit hooks, continuous integration, or scheduled tasks:

```bash
# Format and fix linting issues
claude -p "Fix all linting errors in src/"

# Generate documentation
claude -p "Update API documentation for changes since last commit"

# Run in CI with JSON output
claude --output-format stream-json -p "Verify all tests pass and coverage meets 80%"
```

This enables Claude Code integration into existing development workflows without manual intervention.

**GitHub integration automates pull request reviews.** Install via `/install-github-app` to enable automatic PR analysis. Customize review prompts in `claude-code-review.yml` to enforce project-specific standards. For data science repositories, configure reviews to check statistical methodology, validate data handling, and verify reproducibility requirements.

## Integration patterns for R, R Shiny, and API development

**R projects integrate with Claude Code through terminal-based operation or direct RStudio connection.** The terminal approach runs Claude Code from RStudio's terminal pane, providing full project access while staying in the IDE. Install Node.js 18+, then `npm install -g @anthropic-ai/claude-code` and launch with `claude` from your R project directory. Claude Code automatically recognizes `.Rproj` files and understands R project structure.

The ClaudeR package provides deeper integration using Model Context Protocol. Install from GitHub:

```r
if (!require("devtools")) install.packages("devtools")
devtools::install_github("IMNMV/ClaudeR")

library(ClaudeR)
install_clauder()  # Sets up MCP integration
claudeAddin()      # Starts the server
```

ClaudeR enables Claude Code to execute R code directly in your active RStudio session, see real-time results including plots, access workspace variables, modify document sections, and install packages. This bidirectional communication means Claude can run exploratory commands and see actual outputs rather than guessing behavior. **Security restrictions prevent AI-executed code from running system commands or deleting files**—these safety measures apply only to AI-generated code, not your manual work.

For R package development, CLAUDE.md should explicitly document package structure since models don't naturally understand R packaging conventions:

```markdown
# R Package Structure

## Directory Layout
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
```

Access R package documentation programmatically using the btw package. Configure an MCP server for documentation access:

```bash
claude mcp add -s "user" r-btw -- Rscript -e "btw::btw_mcp_server(tools = 'docs')"
```

Then reference documentation in CLAUDE.md:

```markdown
## Key Dependencies
Read documentation for:
- ?dplyr::filter - Data filtering
- ?ggplot2::ggplot - Plotting fundamentals
- ?tidyr::pivot_longer - Data reshaping
- ?{package} - Package overview
```

Claude Code can query this MCP server to fetch documentation on demand, ensuring accurate usage of packages.

**R Shiny development benefits from incremental, component-based building.** Shiny Assistant (beta from Posit) provides AI-powered Shiny app generation through natural language, supporting both R and Python Shiny. Access at https://gallery.shinyapps.io/assistant/ or self-host with your Claude API key. The architecture runs code in browser via webR/Pyodide (Shinylive), providing three-pane interface: chatbot, code editor, live preview.

Effective Shiny development with AI follows incremental patterns. Start with overall app structure and data familiarization, add components one at a time (UI elements, then interactivity, then advanced features), fix issues before proceeding to next component, and request modularization after core functionality works. Example progression:

1. "Create Shiny app structure for COVID-19 state-level data visualization"
2. "Add summary statistics panel showing total cases and deaths"
3. "Add interactive map showing cases by state"
4. "Add time series chart for selected state"
5. "Split into Shiny modules: data_module.R, map_module.R, stats_module.R"
6. "Generate tests for each module"

Request accessibility from the start: "Create a world-class R Shiny app with excellent accessibility for disabled users. Use bslib best practices, ensure keyboard navigation, provide ARIA labels, and use sufficient color contrast." This prevents retrofitting accessibility later.

Shiny CLAUDE.md files should document reactive programming patterns:

```markdown
# Shiny App Architecture

## Reactive Structure
- Use reactive() for data processing that depends on inputs
- Use observeEvent() for side effects triggered by inputs
- Use renderPlot()/renderTable() etc. for outputs
- Isolate expensive operations with reactiveVal()

## Module Pattern
All modules follow structure:
- `module_ui(id)` - UI function with NS()
- `module_server(id)` - Server function with moduleServer()
- Always use namespacing: ns <- NS(id)

## Best Practices
- Validate inputs before processing
- Use req() to ensure required inputs exist
- Provide loading indicators for slow operations
- Handle errors gracefully with try/catch
- Test reactivity thoroughly

## Common Pitfalls
- ❌ Don't use reactive values outside reactive contexts
- ❌ Don't create infinite reactive loops
- ❌ Don't forget to namespace IDs in modules
- ❌ Don't read reactive values without parentheses: `value()` not `value`
```

**API development with Claude Code follows read-reason-act workflow.** Start by having Claude read relevant files without coding: "Read the API endpoint files in api/ and models in models/schema.py." After Claude understands structure, request planning: "Create a plan for adding JWT authentication to all endpoints." Review the plan, then execute step by step: "Implement step 1 of the authentication plan."

Use subagents for verification: "Use a subagent to verify the authentication flow handles edge cases correctly." This preserves context in main conversation while delegating verification to isolated agent.

For REST API development, organize by resource:

```
api_project/
├── src/
│   ├── routes/
│   │   ├── auth.ts       # Authentication endpoints
│   │   ├── users.ts      # User CRUD
│   │   ├── products.ts   # Product endpoints
│   │   └── orders.ts     # Order management
│   ├── models/           # Data models
│   ├── services/         # Business logic
│   ├── middleware/       # Auth, validation, errors
│   └── utils/
├── tests/
│   ├── integration/      # Full request/response tests
│   └── unit/             # Service/utility tests
├── docs/
│   └── api-spec.yaml     # OpenAPI specification
└── CLAUDE.md
```

CLAUDE.md for APIs should document design principles:

```markdown
# API Architecture

## Design Principles
- RESTful resource-oriented design
- JWT authentication on all endpoints except /auth/login
- JSON:API format for responses
- Semantic versioning (v1, v2, etc. in path)
- Rate limiting: 100 requests/minute per user

## Standard Response Format
```json
{
  "data": { ... },
  "meta": { "timestamp": "...", "version": "1.0" },
  "errors": []
}
```

## Error Handling
- 400: Bad request (validation failed)
- 401: Unauthorized (invalid/missing token)
- 403: Forbidden (valid token, insufficient permissions)
- 404: Not found
- 429: Too many requests (rate limit)
- 500: Internal server error (never expose implementation details)

## Security Requirements
- Helmet.js for security headers
- CORS properly configured
- Input validation with Joi schemas
- Parameterized queries (prevent SQL injection)
- Rate limiting on all endpoints
- Secrets in environment variables only

## Testing Standards
- Jest for unit tests
- Supertest for integration tests
- Test all endpoints (success and error cases)
- Test authentication/authorization
- Minimum 80% coverage
- Mock external dependencies

## Documentation
- OpenAPI 3.0 specification required
- Keep docs/api-spec.yaml updated with code
- Include examples for each endpoint
- Document rate limits and pagination
```

Generate OpenAPI specifications automatically: "Create OpenAPI 3.0 specification for all endpoints in src/routes/, including request/response schemas, authentication schemes, and examples." This ensures documentation stays synchronized with implementation.

**Multi-language integration in polyglot projects uses shared data contracts and language-specific specialization.** The reticulate R package enables Python integration within R sessions, calling Python scripts/modules from R, translating between R and Python objects automatically, and sharing data via accessible attributes (`py$object` in R, `r.object` in Python).

Setup in R:

```r
library(reticulate)
use_python("/path/to/python")  # Or use_virtualenv(), use_condaenv()

# Import Python modules
np <- import("numpy")
pd <- import("pandas")

# Call Python functions
py_run_string("result = sum([1, 2, 3, 4, 5])")
py$result  # Access from R
```

This enables workflows where Python handles ML model training while R performs statistical validation and creates publication-quality visualizations:

```r
library(tidyverse)
library(reticulate)

# R: Load and clean data
data <- read_csv("raw_data.csv") %>%
  filter(!is.na(target)) %>%
  mutate(log_value = log(value + 1))

# Python: Train model
py_run_string("
from sklearn.ensemble import RandomForestClassifier
model = RandomForestClassifier(random_state=42)
model.fit(r.data[['feature1', 'feature2']], r.data['target'])
predictions = model.predict(r.data[['feature1', 'feature2']])
")

# R: Analyze and visualize results
results <- tibble(
  actual = data$target,
  predicted = py$predictions
)

ggplot(results, aes(x = actual, y = predicted)) +
  geom_point(alpha = 0.5) +
  geom_abline(intercept = 0, slope = 1, color = "red") +
  theme_minimal()
```

For polyglot projects, hierarchical CLAUDE.md files with language-specific contexts work best. The root file documents language responsibilities and data flow, while subdirectory CLAUDE.md files provide language-specific details. Document interoperability explicitly:

```markdown
# Root CLAUDE.md

## Language Division
- Python: Data engineering, ML models, API backend
- R: Statistical analysis, visualization, Shiny dashboards
- Data interchange: Parquet format in data/processed/

## Interoperability
- Python → R: Save to Parquet with pandas.to_parquet()
- R → Python: Save to Parquet with arrow::write_parquet()
- Never use CSV for large data (slow and loses types)
- Document data contracts in shared/schemas/

## Cross-Language Testing
- Integration tests verify data contracts
- Run: `make test-python`, `make test-r`, `make test-integration`
```

Configure Claude Code with language-specific skills in `.claude/skills/`:

```
.claude/skills/
├── python-ml/
│   └── SKILL.md        # ML best practices, scikit-learn patterns
├── r-stats/
│   └── SKILL.md        # Statistical analysis, tidyverse conventions
└── data-contracts/
    └── SKILL.md        # Parquet schemas, validation patterns
```

These skills activate on-demand, providing specialized guidance without consuming base context.

## Transitioning from system prompts to sophisticated documentation

**The evolution from append-system-prompt to documentation-based approaches dramatically improves scalability and maintainability.** System prompts apply globally and consume tokens in every interaction. CLAUDE.md files load once as first user message, targeting project-specific context. Skills load on-demand with progressive disclosure, providing specialized capabilities only when needed.

Research from LangChain shows **CLAUDE.md plus MCP integration outperforms MCP alone by 15-20%**. Raw documentation access fills context faster without proportional benefit. High-quality condensed information (CLAUDE.md) combined with on-demand detailed lookup (MCP) achieves optimal results. The pattern: encode base knowledge in concise CLAUDE.md, reference detailed documentation through MCP, and use Skills for recurring specialized workflows.

Migration follows four phases:

**Phase 1: Inventory existing system prompts.** Document all `--append-system-prompt` usage. Categorize content: coding conventions (candidates for CLAUDE.md), command documentation (belongs in CLAUDE.md), specialized workflows (extract to commands), recurring procedures (convert to Skills), global behavior (keep as system prompt).

**Phase 2: Create initial CLAUDE.md.** Start with essentials: tech stack, key commands (build, test, lint), directory structure, critical conventions. Add iteratively based on repeated instructions. Test by removing corresponding system prompt sections and verifying Claude still exhibits correct behavior. Keep under 100 lines initially—resist temptation to dump everything.

**Phase 3: Extract specialized content.** Identify sections over 100 tokens in CLAUDE.md that represent repeatable, specialized tasks. Convert to Skills with progressive disclosure: description in CLAUDE.md, full details in skill file. Create slash commands for workflows requiring step-by-step execution. Configure MCP servers for external system access.

**Phase 4: Optimize and maintain.** Monitor which sections Claude references most frequently. Update based on team feedback and new patterns emerging. Use Claude Code's `/memory` command to add information during sessions—items added with `#` prefix automatically update CLAUDE.md. Version control CLAUDE.md alongside code for team consistency.

Decision matrix for content placement:

**System Prompts** for: global tone/personality, security constraints that must never be violated, high-level guardrails, persistent behavioral modifications.

**CLAUDE.md** for: project-specific conventions, development commands, directory structure, architecture patterns, domain context, methodological preferences.

**Skills** for: repeatable specialized workflows, multi-step procedures, cross-project capabilities, token-heavy instructions that apply occasionally.

**Commands** for: specific task templates, parameterized workflows, frequently used operations.

**MCP Servers** for: external system access (databases, APIs, tools), dynamic information lookup, real-time data access.

Before migration:
```bash
claude --append-system-prompt "You are a data scientist. Always validate data first. 
Use pandas for manipulation. Create matplotlib visualizations with seaborn styling. 
Include comprehensive error handling. Document all analysis decisions. Use type hints 
for Python functions. Run pytest before committing. Check statistical assumptions..." 
# 500+ tokens every session
```

After migration:
```markdown
# CLAUDE.md (~200 tokens, loaded once)
# Data Science Project

## Stack: Python 3.11, pandas, scikit-learn, pytest
## Commands: pytest, jupyter lab, python -m src.pipeline.run
## Conventions: Type hints required, median imputation for missing values
## Skills available: /data-validation, /statistical-analysis, /ml-experiment
```

Benefits: 60-80% token reduction, version-controlled documentation, team-wide consistency, right context at right time, scales without performance degradation.

**The Skills system represents cutting-edge modular expertise.** Skills differ from commands in four ways: progressive disclosure (only full details load when needed), code execution capability (can include Python scripts for reliable operations), automatic coordination (multiple skills work together), and cross-platform portability (work in Claude.ai, Claude Code, and API).

Create a data profiling skill:

```markdown
---
name: data-profiling
description: Comprehensive data quality assessment and statistical profiling 
  for datasets. Automatically detects data issues and generates detailed reports.
---

# Data Profiling Skill

This skill analyzes datasets to generate comprehensive quality reports.

## Capabilities
- Missing value analysis with patterns (MCAR, MAR, MNAR)
- Data type inference and validation
- Statistical summaries (descriptive statistics)
- Outlier detection (IQR method, Z-scores)
- Distribution analysis and normality tests
- Correlation analysis
- Cardinality assessment
- Duplicate detection

## Usage
"Profile the customer_data.csv file"
"Generate data quality report for sales DataFrame"
"Check data quality before modeling"

## Workflow
1. Load dataset and infer types
2. Calculate completeness metrics
3. Detect outliers by column
4. Test distributions
5. Analyze correlations
6. Generate summary report
7. Flag quality issues

[Detailed implementation instructions...]
```

Reference skills in CLAUDE.md: "Available skills: data-profiling (use for quality checks), statistical-testing (use for hypothesis tests), ml-pipeline (use for model training)." Claude loads skill details only when relevant to current task, maintaining lean context.

Simon Willison's insight captures the genius: "Skills are conceptually extremely simple but incredibly powerful. The genius is progressive disclosure—each skill only takes a few dozen tokens until needed, making them far more scalable than monolithic prompts." This architecture enables hundreds of specialized capabilities without context bloat.

## Repository examples and community resources worth studying

**The awesome-claude-code repository curates 100+ exemplary resources.** Access at github.com/hesreallyhim/awesome-claude-code for commands, CLAUDE.md files, workflows, and tools organized by category. Notable data science examples include `/load_dango_pipeline` for ML training context, gene ontology annotation commands, and data validation utilities. The repository demonstrates command organization patterns, namespace usage, and language-specific configurations.

**VoltAgent/awesome-claude-code-subagents provides 100+ production-ready agents** with battle-tested definitions following industry standards. Data science agents include data-analyst (insights and visualization), data-engineer (pipeline architecture), data-scientist (analytics expert), ml-engineer (model training), mlops-engineer (deployment), nlp-engineer (text analysis), and database-optimizer (performance tuning). Each agent file includes role definition, MCP tool integration, communication protocols, and implementation workflows. Study these for agent design patterns.

**The claude-flow repository (github.com/ruvnet/claude-flow) specializes in orchestration.** It provides agent coordination platforms with native Claude Code support via MCP protocol. Data science features include pre-configured swarm initialization, parallel agent execution for analysis workflows, statistical analysis coordination, and data pipeline templates. The repository demonstrates multi-agent communication patterns and complex workflow orchestration.

**Template collections at aitmpl.com serve as de facto package manager** with 400+ components across six categories: agents (specialized personas), commands (task configurations), settings (environment configs), hooks (lifecycle automation), MCPs (protocol integrations), and templates (complete setups). Browse the professional template gallery for inspiration and download components directly.

Real-world case studies demonstrate practical implementation. The urban-sounds audio ML project refactoring (documented at sjoerddehaan.com) shows Claude Code consolidating scattered Jupyter notebooks, centralizing configuration, and applying software engineering principles to research code. The entire transformation took 30 minutes, creating single source of truth for paths, standardized loading, environment-aware configuration, and comprehensive documentation.

The SQL analysis automation case (documented at pub.towardsai.net) evolved through multiple attempts. Initial vague requests generated incorrect queries. Adding database schema documentation, example queries, and explicit task sequences succeeded. The final pattern: comprehensive context in CLAUDE.md, template queries, explicit task ordering, MCP SQL servers for secure access, and breaking workflows across sessions for complex analyses.

Customer churn prediction tutorials (from Dataquest) demonstrate complete pipeline development: configurable data loading with validation, automated EDA script generation, preprocessing pipelines with error handling, and comprehensive test generation. The workflow produces production-ready code in hours instead of days.

**Successful repositories share four characteristics.** Clear documentation with setup instructions and usage patterns. Structured organization with logical layout and consistent conventions. Production-ready patterns including error handling, testing strategies, and configuration management. Modular, reusable components with parameterized commands shareable across projects.

Critical success factors differ by scale. Individual data scientists should start with small low-risk tasks, build personal prompt libraries, use `/clear` frequently for context management, maintain detailed CLAUDE.md with data schemas, and leverage extended thinking for statistical debugging. Teams benefit from sharing CLAUDE.md via git, creating team command libraries, standardizing directory structure, using hierarchical CLAUDE.md for complexity, and documenting analytical methodologies. Complex projects require separating planning from execution, using external files as checklists, breaking workflows into multiple sessions, leveraging subagents for parallel work, and maintaining living documentation of decisions.

Common pitfalls include insufficient context (Claude doesn't know your domain), overly complex requests (break into focused steps), accepting output without review (always validate statistics), context bloat (clear frequently, reference only relevant files), and vague prompts (be specific about methods and requirements).

## Implementation roadmap: From setup to sophisticated workflows

**Week 1: Foundation setup.** Run `/init` in your main data science project. This interactive command generates tailored CLAUDE.md based on project structure. Review and refine the generated file, adding data schemas, key commands, and critical conventions. Create basic `.gitignore` entry for `.claude/settings.local.json` to exclude personal settings. Document environment activation (conda, venv, or micromamba commands) and primary workflows (run tests, start notebooks, execute pipelines).

**Week 2: Custom commands.** Identify three workflows you repeat most frequently. Convert to slash commands in `.claude/commands/`. For data science, start with `/eda` (exploratory data analysis), `/test-data` (data validation), and `/generate-report` (analysis documentation). Test each command, refine based on results. Share effective commands with team by committing to version control.

**Week 3: Specialized subagents.** Create two subagents addressing your most common needs. Start with data-quality-checker (validates datasets, detects issues) and one domain-specific agent (statistical-analyst, ml-engineer, or visualization-specialist depending on focus). Test explicit invocation first ("Use data-quality-checker to validate this dataset"), then let automatic delegation activate. Refine descriptions based on delegation accuracy.

**Week 4: MCP integration.** If you access databases or external systems regularly, configure appropriate MCP servers. Start read-only for safety. For database access, test query generation and review before execution. Gradually expand permissions as confidence grows. Document MCP server configuration in project README for team awareness.

**Week 5-8: Advanced patterns.** Add hooks for automatic formatting or test execution. Create hierarchical CLAUDE.md files for subdirectories. Extract Skills from repetitive CLAUDE.md sections. Implement complex multi-agent workflows for end-to-end analysis pipelines. Share successful patterns with broader team.

For R package developers, special considerations apply. Install ClaudeR for direct RStudio integration. Configure btw MCP server for documentation access. Document package structure explicitly in CLAUDE.md since models don't naturally understand R packaging. Use Git extensively—no native undo in Claude Code. Pause before test runs to interpret failures yourself. Reserve Claude Code for 15+ minute refactoring tasks where context switching overhead justifies AI assistance. Typical usage costs $5-10 per developer per day for intermittent use, $30-100+ for intensive sessions.

For Shiny developers, use Posit's Shiny Assistant for rapid prototyping. Build incrementally, one component at a time. Request modularization after core features work. Generate tests separately for each module. Include accessibility requirements from start. Document reactive programming patterns clearly. Test interactivity thoroughly before adding complexity.

For API developers, follow read-reason-act workflow. Have Claude read relevant files before planning. Use subagents for verification to preserve context. Generate OpenAPI specifications alongside implementation. Test authentication and error handling comprehensively. Document security requirements explicitly.

Team adoption requires staged rollout. Individual contributors start with personal configurations (`~/.claude/agents`, `CLAUDE.local.md`), experiment with workflows, and document successes. Team rollout creates shared `.claude/agents` in repositories, establishes CLAUDE.md standards, shares slash commands, holds demo sessions, and version controls all configurations. Enterprise considerations include project-scoped MCP servers for security, documented data access patterns, carefully reviewed auto-accept permissions, cross-team CLAUDE.md standards, and shared skill libraries.

## Conclusion: Documentation as infrastructure for intelligent development

**Claude Code transforms data science productivity through intelligent context management rather than raw capability.** The difference between frustrating and phenomenal experiences lies in configuration quality, not model selection. Invest time in comprehensive CLAUDE.md files, create focused subagents encoding domain expertise, build libraries of portable commands, configure appropriate MCP integrations, and maintain documentation as living artifacts.

The three-layer system—CLAUDE.md for base context, commands for workflows, subagents for specialization—scales from individual projects to enterprise deployments. Start simple with basic CLAUDE.md, add commands for frequent operations, create subagents when clear specialization emerges, and extract Skills when patterns solidify. This progressive enhancement prevents overwhelming complexity while enabling sophisticated capabilities.

Data science particularly benefits from Claude Code's context understanding. Statistical methodology documentation ensures appropriate test selection. Data schema specification prevents incorrect assumptions. Hierarchical project organization enables polyglot workflows. Specialized subagents encode disciplinary best practices. The combination produces code that follows field standards automatically.

Real-world implementations demonstrate 5-10x productivity improvements with proper configuration. The key: treat Claude Code as collaborative partner requiring clear communication, not magic solution expecting perfection. Provide comprehensive context upfront, use specific instructions over vague requests, validate outputs rigorously, clear context frequently, and iterate based on results.

Community resources continue expanding. Awesome-claude-code curates best practices. VoltAgent maintains production-ready agents. Claude-flow provides orchestration templates. aitmpl.com offers components marketplace. Official Anthropic documentation details features comprehensively. These resources accelerate learning and prevent reinventing patterns.

The future of data science development combines human expertise with AI capability through sophisticated configuration. Those who invest in documentation infrastructure—CLAUDE.md files, command libraries, specialized agents, MCP integrations—unlock transformative productivity while maintaining code quality and methodological rigor. Start today with `/init` in your main project, then enhance iteratively as needs emerge.

Your competitive advantage comes not from having Claude Code, but from configuring it excellently. The patterns documented here provide foundation. Your domain expertise, project requirements, and team workflows shape the specifics. Build your documentation infrastructure deliberately, and Claude Code becomes force multiplier for data science work.