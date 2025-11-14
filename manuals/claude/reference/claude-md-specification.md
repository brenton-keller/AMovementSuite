# CLAUDE.md Specification

Complete reference for CLAUDE.md syntax, options, and features.

## Overview

CLAUDE.md files provide automatic project memory that loads when Claude Code starts. They are markdown files containing project-specific context, conventions, and essential information.

## File Locations

| Location | Scope | Load Order | Version Control |
|----------|-------|------------|-----------------|
| `.claude/CLAUDE.md` | Project-level (preferred) | 3rd | Yes - commit to repo |
| `CLAUDE.md` | Project root (alternative) | 3rd | Yes - commit to repo |
| `~/.claude/CLAUDE.md` | User-level (all projects) | 2nd | No - personal preferences |
| Enterprise-level | Organization-wide | 1st | Varies by setup |

**Load hierarchy:** Enterprise → User → Project. Project-level overrides take precedence for conflicts.

## File Structure

### Recommended Sections

```markdown
# Project Name

## Tech Stack
[Languages, frameworks, key dependencies]

## Directory Structure
[Project organization explanation]

## Development Commands
[Build, test, run commands]

## Coding Standards
[Project-specific conventions]

## Critical Constraints
[Things that must never change]

## Data Schemas (for data science)
[Column names, types, expected ranges]

## Architecture Decisions
See docs/architecture/decisions/

## Protected Areas
- ❌ [Things AI should never modify]
```

## Import System

### Basic Import Syntax

```markdown
## Architecture
See docs/architecture/overview.md

## Detailed Patterns
docs/patterns/database-connections.md
```

**Note:** Use plain file paths only. Do NOT use `@` syntax in CLAUDE.md files.

### Import Examples

| Pattern | Purpose | When to Use |
|---------|---------|-------------|
| `docs/architecture/decisions.md` | Reference external doc | Keep CLAUDE.md lean |
| `~/.claude/personal-preferences.md` | User preferences | Personal coding style |
| `docs/schemas/data-definitions.md` | Data schemas | Data science projects |
| `docs/troubleshooting/common-errors.md` | Error solutions | Debugging context |

### Progressive Disclosure Pattern

```markdown
# Main CLAUDE.md (concise, <100 lines)

## Quick Reference
- Build: `npm run build`
- Test: `npm test`
- Deploy: `npm run deploy`

## Detailed Documentation
For architecture details: docs/architecture/
For API specs: docs/api/
For troubleshooting: docs/troubleshooting/
```

## Size Recommendations

| Project Type | Target Size | Maximum Size | Rationale |
|--------------|-------------|--------------|-----------|
| Small projects | 50-100 lines | 200 lines | Base context always loaded |
| Medium projects | 100-150 lines | 300 lines | Use imports for details |
| Large projects | 150-200 lines | 400 lines | Heavy use of imports required |
| Enterprise | 200-250 lines | 500 lines | Hierarchical structure essential |

**Key principle:** Every line consumes context tokens in each interaction. Be ruthlessly concise.

## Content Guidelines

### What to Include

**Essential (always include):**
- Tech stack and versions
- Key development commands
- Non-obvious project conventions
- Critical constraints
- Protected areas (never modify)

**For Data Science:**
- Data schemas (columns, types, ranges)
- Statistical methodology preferences
- Missing value handling strategy
- Random seed conventions

**For Multi-language:**
- Language responsibilities
- Data interchange formats
- Cross-language conventions

### What to Exclude

**Do NOT include:**
- General programming knowledge
- Language syntax basics
- Framework documentation (link instead)
- Detailed implementation (reference code)
- Changelog information

### Conciseness Techniques

| Instead of | Use |
|------------|-----|
| "We use Python 3.11 with pandas for data manipulation and scikit-learn for machine learning" | "Stack: Python 3.11, pandas, scikit-learn" |
| "When you encounter missing values in numerical columns, you should use median imputation" | "Missing values: median imputation for numerical" |
| Full file listings | "See src/ for structure" |
| Detailed API docs | "API: docs/api/reference.md" |

## Hierarchical CLAUDE.md

### Directory-Specific Context

```
project/
├── CLAUDE.md              # Root context
├── python/
│   └── CLAUDE.md         # Python-specific context
└── r/
    └── CLAUDE.md         # R-specific context
```

**Behavior:** When Claude navigates to `python/`, it automatically loads `python/CLAUDE.md` in addition to root context.

### Root CLAUDE.md Example

```markdown
# Multi-Language Analytics Platform

## Language Responsibilities
- Python: Data engineering, ML models, API services
- R: Statistical analysis, reporting, Shiny dashboards

## Data Flow
Python ETL → data/processed/*.parquet → R analysis → reports/

## Cross-Language Conventions
- Use pathlib (Python) / here package (R) for paths
- UTC timezone for all timestamps
- Snake_case for variable names
- Maximum line length: 100 characters

## Common Commands
- `make test-all`: Run tests across all languages
- `make setup`: Initialize all environments
```

### Subdirectory CLAUDE.md Example

```markdown
# Python Module Context

## Environment
- Python 3.11+ required
- Activate: `source venv/bin/activate`
- Dependencies: `pip install -r requirements.txt`

## Code Conventions
- Type hints required for all functions
- Google-style docstrings
- Use pathlib, never os.path
- Set random seeds to 42

## Testing
- Unit tests mirror src/ structure
- Use fixtures from tests/conftest.py
- Minimum 80% coverage required
```

## Markers and Symbols

### Protected Areas

```markdown
## Protected Areas
- ❌ Never modify raw data files in data/raw/
- ❌ Don't duplicate code between languages
- ❌ No hardcoded credentials (use environment variables)
```

### Status Indicators

```markdown
## Status
🚧 Prototype - Active development
✅ Production - Stable
⚠️  Deprecated - Migrate to new approach
```

### Priority Markers

```markdown
## Critical Constraints
🔴 MUST: Always validate data before processing
🟡 SHOULD: Prefer pandas over raw loops
🟢 NICE: Add type hints for better IDE support
```

## Common Patterns

### Data Science Project

```markdown
# Project Name

## Stack
Python 3.11, pandas, scikit-learn, pytest

## Data Schema
customers.csv:
- customer_id: int (primary key)
- signup_date: datetime (UTC)
- total_spend: float (USD, non-negative)
- segment: str (bronze|silver|gold)

## Statistical Conventions
- Missing values: median imputation for numerical
- Multiple comparisons: Bonferroni correction
- Random seed: 42
- Significance level: α = 0.05

## Commands
- `pytest tests/`
- `jupyter lab`
- `python -m src.pipeline.run`

## Protected
- ❌ Never modify data/raw/
```

### API Project

```markdown
# API Project

## Stack
Node.js 20, Express, PostgreSQL, Jest

## API Design
- RESTful resource-oriented
- JWT authentication (all endpoints except /auth/login)
- JSON:API response format
- Rate limit: 100 requests/minute per user

## Error Codes
- 400: Bad request (validation failed)
- 401: Unauthorized (invalid/missing token)
- 403: Forbidden (insufficient permissions)
- 404: Not found
- 500: Internal error (never expose details)

## Commands
- `npm test`
- `npm run dev`
- `npm run deploy`

## Security
- ❌ No credentials in code
- ✅ Parameterized queries only
```

### Multi-Module Monorepo

```markdown
# Monorepo Project

## Structure
- `apps/`: Deployable applications
- `libs/`: Shared libraries
- `tools/`: Development scripts

## Build
- Tool: Nx
- Commands: `nx build [app]`, `nx test [project]`

## Conventions
- Each lib has public API in index.ts
- No circular dependencies
- Shared types in libs/shared/types

## Module Boundaries
Enforce with `nx.json` module boundaries
```

## Common Pitfalls

| Problem | Wrong | Right |
|---------|-------|-------|
| Too verbose | 500+ line CLAUDE.md with full API docs | <200 lines, link to docs/ |
| General knowledge | "Python uses 0-based indexing" | Only project-specific info |
| Outdated info | Never updated CLAUDE.md | Update with code changes |
| Wrong syntax | Using `@` syntax for file references | Plain file paths only |
| Missing constraints | No protected areas defined | Clear ❌ markers for critical areas |
| No structure | Wall of text | Clear sections with headers |

## Memory System Integration

### Adding to Memory During Session

```
# During a conversation with Claude:
/memory Project uses semantic versioning #
/memory API rate limit is 1000 req/min #
```

**Note:** Items added with `#` prefix automatically update CLAUDE.md.

### Compaction

When conversation gets long:
```
/compact Keep everything after feature engineering discussion
```

This preserves recent context while clearing early exploration.

## Version Control

### What to Commit

**Always commit:**
- `.claude/CLAUDE.md` (project conventions)
- `.claude/commands/` (team commands)
- `.claude/agents/` (team agents)

**Never commit:**
- `.claude/settings.local.json` (personal settings)
- `~/.claude/` (user-level configs)

### .gitignore Recommended

```gitignore
# Personal Claude settings
.claude/settings.local.json
.claude/local/

# User-level (not in project)
# ~/.claude/ already outside repo
```

## Best Practices Summary

1. **Keep it concise** - Under 200 lines when possible
2. **Use imports** - Link to detailed docs instead of inlining
3. **Non-obvious only** - Skip general programming knowledge
4. **Update regularly** - Keep in sync with codebase
5. **Clear sections** - Organize with headers
6. **Mark protected areas** - Use ❌ for constraints
7. **Hierarchical context** - Subdirectory CLAUDE.md for specifics
8. **Version control** - Commit project-level, not user-level
9. **Progressive disclosure** - Overview in CLAUDE.md, details in imports
10. **Data schemas** - Essential for data science projects

## Validation Checklist

- [ ] File is under 200 lines (or well-justified if longer)
- [ ] Contains only project-specific information
- [ ] Uses plain file paths (no `@` syntax)
- [ ] Includes key development commands
- [ ] Documents critical constraints with ❌
- [ ] References detailed docs with file paths
- [ ] Contains data schemas (if data science project)
- [ ] Version controlled in repository
- [ ] Updated within last month
- [ ] Reviewed by team member

## See Also

- **Command Specification**: command-specification.md
- **Agent Specification**: agent-specification.md
- **File Path Syntax**: file-path-syntax.md
- **Project Structures**: project-structures.md
