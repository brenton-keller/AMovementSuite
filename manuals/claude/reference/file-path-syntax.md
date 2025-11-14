# File Path Syntax Reference

Complete guide to referencing files in Claude Code - emphasizing plain file paths without special syntax.

## Critical Rule: NO @ Syntax

**Important:** Claude Code does NOT use `@` syntax for file references in CLAUDE.md files, commands, or agent definitions.

| Context | Wrong | Right |
|---------|-------|-------|
| CLAUDE.md | `@docs/architecture/overview.md` | `docs/architecture/overview.md` |
| Command | `See @src/utils/helpers.py` | `See src/utils/helpers.py` |
| Agent | `Read @data/schema.json` | `Read data/schema.json` |
| Conversation | `@src/main.py` | `src/main.py` (or use Read tool) |

**The `@` syntax is for conversation context in Claude.ai web interface, NOT for Claude Code.**

## File Path Basics

### Absolute vs Relative Paths

**Relative paths (preferred in CLAUDE.md):**
```markdown
## Documentation
See docs/architecture/decisions.md
See ../shared/types.ts
See ./local-config.md
```

**Absolute paths (for tools and commands):**
```bash
# In Bash tool
cat /absolute/path/to/file.txt

# Windows
C:\Workspace\project\file.txt
```

**Best practice:** Use relative paths in documentation, absolute paths in tool execution.

### Path Separators

| Platform | Separator | Example |
|----------|-----------|---------|
| Linux/Mac | `/` | `src/modules/user/index.ts` |
| Windows | `\` | `src\modules\user\index.ts` |
| Cross-platform | `/` | `src/modules/user/index.ts` (works on Windows too) |

**Recommendation:** Use forward slash `/` for cross-platform compatibility.

## In CLAUDE.md Files

### Basic File References

```markdown
# Project Documentation

## Architecture
See docs/architecture/overview.md for system design.

## API Reference
Full API documentation: docs/api/reference.md

## Data Schemas
Customer schema: docs/schemas/customer.json
Product schema: docs/schemas/product.json

## Troubleshooting
Common errors: docs/troubleshooting/common-issues.md
```

**How it works:** Claude reads these as plain text and can navigate to files if you ask.

### Linking to Code

```markdown
## Key Files
- Main application: src/main.py
- Configuration: config/settings.json
- Database models: src/models/user.py

## Patterns
Database connection pattern: src/db/connection.py
Authentication pattern: src/auth/jwt.py
```

### User-Level File References

Reference files in user directory:
```markdown
## Personal Preferences
See ~/.claude/personal-preferences.md

## Shared Utilities
Personal commands: ~/.claude/commands/
```

**Note:** `~` expands to user home directory on Linux/Mac.

## In Slash Commands

### Referencing Files in Command Instructions

```markdown
---
description: Run tests based on project configuration
allowed-tools: Bash, Read
---

# Test Command

## Detection

Check for these files to determine project type:
- Python: pytest.ini or tests/*.py
- R: testthat/ or tests/testthat/
- JavaScript: jest.config.js or test/*.js

## Usage

If pytest.ini exists, run Python tests.
If testthat/ exists, run R tests.
```

### Reading Files in Commands

```markdown
---
description: Validate configuration
allowed-tools: Read
---

# Validate Config

Read and validate: $1

## Steps

1. Read file: $1
2. Parse as JSON
3. Validate schema against docs/schemas/config-schema.json
4. Report any violations
```

**Usage:** `/validate-config config/settings.json`

## In Agent Definitions

### Agent Instructions with File Paths

```markdown
---
name: code-reviewer
description: Reviews code quality
tools: Read
---

You are a code reviewer.

## Reference Materials

Check code against standards in:
- docs/coding-standards.md
- docs/architecture/patterns.md

## Process

1. Read the code file provided
2. Compare against standards
3. Generate review report
```

### Agent Reading Patterns

```markdown
---
name: data-validator
description: Validates datasets
tools: Read, Write
---

## Validation Process

1. Read schema from docs/schemas/[dataset-name].json
2. Read data from data/raw/[dataset-name].csv
3. Validate each column against schema
4. Write report to reports/validation/[dataset-name]-report.md
```

## In Conversation

### Asking Claude to Read Files

**Plain references (Claude navigates):**
```
Please review the code in src/main.py
```

**Explicit Read tool:**
```
Read src/main.py and check for security issues
```

**Multiple files:**
```
Read these files:
- src/models/user.py
- src/models/product.py
- tests/test_models.py
```

### Writing/Editing Files

**Write new file:**
```
Create a new file at src/utils/validator.py with validation functions
```

**Edit existing:**
```
Update src/config.json to add the new API endpoint
```

**Multiple operations:**
```
1. Read src/main.py
2. Create tests/test_main.py
3. Update docs/api.md with new endpoints
```

## Path Patterns by Use Case

### Documentation Links

```markdown
# In CLAUDE.md

## Architecture
- System overview: docs/architecture/overview.md
- Decision records: docs/architecture/adr/
- Database design: docs/architecture/database.md

## Development
- Setup guide: docs/development/setup.md
- Testing strategy: docs/development/testing.md
- Deployment: docs/development/deployment.md
```

### Data File References

```markdown
# In CLAUDE.md - Data Science Project

## Data Files
- Raw data: data/raw/customers.csv
- Processed: data/processed/customers.parquet
- Schemas: data/schemas/customer-schema.json

## Data Flow
1. Load from data/raw/
2. Process and validate
3. Save to data/processed/
4. Document in docs/data-dictionary.md
```

### Configuration Files

```markdown
# In CLAUDE.md

## Configuration
- App config: config/app.json
- Database config: config/database.json
- Environment variables: .env.example

## Secrets
- ❌ Never commit config/secrets.json
- ✅ Use environment variables
```

### Test Files

```markdown
# In CLAUDE.md

## Testing
- Test directory: tests/
- Unit tests: tests/unit/
- Integration tests: tests/integration/
- Test fixtures: tests/fixtures/
- Test configuration: pytest.ini or tests/testthat/
```

## Platform-Specific Considerations

### Windows Paths

**In CLAUDE.md (use forward slash):**
```markdown
See docs/architecture/overview.md
```

**In Bash tool (may need backslash):**
```bash
# Forward slash usually works
cat C:/Workspace/project/file.txt

# Backslash if required
type C:\Workspace\project\file.txt
```

**PowerShell:**
```powershell
Get-Content C:\Workspace\project\file.txt
```

### Linux/Mac Paths

**Standard Unix paths:**
```markdown
See /home/user/project/docs/readme.md
See ~/project/docs/readme.md
```

**Tilde expansion:**
```markdown
# User home directory
~/.claude/CLAUDE.md

# Expands to
/home/username/.claude/CLAUDE.md
```

### Cross-Platform Compatibility

**Best practices:**
1. Use forward slash `/` in CLAUDE.md (works on all platforms)
2. Use relative paths when possible
3. Avoid hardcoding absolute paths
4. Use environment variables for platform-specific paths

```markdown
# Good (cross-platform)
See docs/setup.md
Read src/config.json

# Avoid (platform-specific)
See C:\Users\John\project\docs\setup.md
Read /home/john/project/src/config.json
```

## Special Directories

### Project Root

```markdown
# In CLAUDE.md at project root
See README.md
See LICENSE
See .gitignore
```

### Subdirectory Context

```markdown
# In python/CLAUDE.md (subdirectory)

## Python Module
Current module: python/

## Files
- Main: src/main.py
- Tests: tests/
- Config: pyproject.toml

## Root Documentation
See ../docs/architecture/  (relative to root)
```

### User Home Directory

```markdown
# User-level CLAUDE.md at ~/.claude/CLAUDE.md

## Personal Settings
Commands: ~/.claude/commands/
Agents: ~/.claude/agents/
Settings: ~/.claude/settings.json

## Project References
Current project: (determined by working directory)
```

## Common Patterns

### Pattern 1: Hierarchical Documentation

```markdown
# Root CLAUDE.md

## Documentation
- Architecture: docs/architecture/
  - Overview: docs/architecture/overview.md
  - ADRs: docs/architecture/adr/
  - Database: docs/architecture/database.md
- API: docs/api/
  - Reference: docs/api/reference.md
  - Examples: docs/api/examples.md
- Development: docs/development/
  - Setup: docs/development/setup.md
  - Testing: docs/development/testing.md
```

### Pattern 2: Source Code Structure

```markdown
# In CLAUDE.md

## Source Code
- Main: src/
  - Modules: src/modules/
  - Utils: src/utils/
  - Config: src/config/
- Tests: tests/
  - Unit: tests/unit/
  - Integration: tests/integration/
```

### Pattern 3: Multi-Language References

```markdown
# Root CLAUDE.md

## Python Code
See python/src/
Python docs: python/README.md

## R Code
See r/R/
R docs: r/README.md

## Shared
Data contracts: shared/schemas/
Shared docs: docs/interop.md
```

### Pattern 4: External Tool Configuration

```markdown
# In CLAUDE.md

## Build Configuration
- Node: package.json
- Python: pyproject.toml
- R: DESCRIPTION, renv.lock
- Docker: Dockerfile, docker-compose.yml

## CI/CD
- GitHub Actions: .github/workflows/
- Pre-commit: .pre-commit-config.yaml
```

## File Path Pitfalls

### Common Mistakes

| Mistake | Problem | Solution |
|---------|---------|----------|
| `@docs/file.md` | Wrong syntax | `docs/file.md` |
| `docs\file.md` (in CLAUDE.md) | Windows-only | `docs/file.md` |
| `/absolute/path` (in CLAUDE.md) | Not portable | Use relative paths |
| `./././docs/file.md` | Overly complex | `docs/file.md` |
| No path at all | Claude doesn't know where | Always specify location |

### Ambiguous References

**Bad:**
```markdown
See the architecture document.
Check the schema file.
```

**Good:**
```markdown
See docs/architecture/overview.md
Check data/schemas/customer-schema.json
```

### Missing Context

**Bad:**
```markdown
## Files
- main.py
- config.json
```

**Good:**
```markdown
## Files
- Main application: src/main.py
- Configuration: config/config.json
```

## Validation

### Path Reference Checklist

- [ ] No `@` syntax used
- [ ] Forward slashes `/` for cross-platform compatibility
- [ ] Relative paths used in documentation
- [ ] Paths actually exist in repository
- [ ] Specific file names (not just directory)
- [ ] Clear context provided
- [ ] Works from project root

### Testing File References

**Manual verification:**
1. Navigate to project root
2. Check if path exists: `ls docs/architecture/overview.md`
3. Try referencing in Claude Code
4. Verify Claude can read the file

## Quick Reference Table

| Context | Syntax | Example |
|---------|--------|---------|
| CLAUDE.md reference | Plain path | `docs/setup.md` |
| User home | `~/` prefix | `~/.claude/commands/` |
| Relative parent | `../` | `../docs/api.md` |
| Current directory | `./` (optional) | `./README.md` or `README.md` |
| Subdirectory | Plain path | `src/modules/user/` |
| Windows absolute | Drive letter | `C:/Workspace/project/` |
| Unix absolute | Leading `/` | `/home/user/project/` |
| Conversation Read | Plain mention | "Read src/main.py" |
| Command argument | `$1`, `$ARGUMENTS` | File path via argument |

## Best Practices Summary

1. **Never use @ syntax** - Plain file paths only
2. **Use forward slashes** - Cross-platform compatible
3. **Prefer relative paths** - Portable across systems
4. **Be specific** - Full path, not just directory
5. **Provide context** - Explain what file contains
6. **Verify existence** - Ensure files actually exist
7. **Document structure** - Explain directory organization
8. **Use consistent style** - Maintain path format throughout

## See Also

- **CLAUDE.md Specification**: claude-md-specification.md
- **Command Specification**: command-specification.md
- **Agent Specification**: agent-specification.md
- **Project Structures**: project-structures.md
