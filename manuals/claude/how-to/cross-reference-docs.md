# How to Reference Documentation Without @ Syntax

## Problem

You need Claude Code to access additional documentation during a session, but:

- Using @ syntax for every reference clutters conversations
- Loading too many files at once fills context quickly
- You want progressive disclosure (load details only when needed)
- @ syntax doesn't work in some contexts (CLAUDE.md, slash commands)
- You need to reference external documentation clearly

**Common scenarios:**
- "Read the API documentation in docs/api/"
- "Check the architecture decisions before proceeding"
- "Reference the database schema documentation"
- "Look at the pattern documentation for examples"

## Solution

Use plain file paths strategically and let Claude read files on-demand when needed. This approach maintains lean context while keeping detailed information accessible.

## Key Principle

**@ syntax is for autocomplete and interactive selection. Plain file paths work everywhere and are more explicit.**

## Methods for Referencing Documentation

### Method 1: Plain File Path References (Recommended)

**In conversation:**
```
Read the API specification at docs/api/openapi.yaml before implementing
```

**In CLAUDE.md:**
```markdown
## Architecture Decisions
See architecture/decisions/ for ADRs

## API Documentation
Full spec: docs/api-spec.yaml
Database schema: docs/schemas/database.md

## Patterns
Standard patterns: docs/patterns/
Database connection: docs/patterns/database-connection.md
Testing: docs/patterns/test-structure.md
```

**In slash commands:**
```markdown
## Additional Context

Before implementing, read:
- docs/api/endpoints.md - API endpoints specification
- docs/patterns/authentication.md - Authentication patterns
```

**Why this works:**
- Claude can read these files when needed
- Doesn't consume context upfront
- Works in all contexts (CLAUDE.md, commands, conversation)
- More explicit than @ syntax

### Method 2: Structured References in CLAUDE.md

**Pattern documentation:**
```markdown
## Code Patterns

When implementing database connections, follow:
docs/patterns/database-connection.md

When writing tests, reference:
docs/patterns/test-structure.md

For error handling, see:
docs/patterns/error-handling.md
```

**Architecture references:**
```markdown
## Architecture

Current decisions documented in:
architecture/decisions/

Key ADRs:
- architecture/decisions/0001-modular-monolith.md - Overall structure
- architecture/decisions/0003-parquet-format.md - Data interchange
- architecture/decisions/0005-testing-strategy.md - Test approach
```

**Schema documentation:**
```markdown
## Data Schemas

Main schemas documented in:
docs/schemas/

Customer data: docs/schemas/customers.md
Transaction data: docs/schemas/transactions.md
Product data: docs/schemas/products.md
```

### Method 3: Inline Brief + Reference Detail

Keep essentials inline, reference details:

```markdown
## Database Connection

**Pattern:** Use connection pooling with automatic retry

Connection details:
- Host: Environment variable DB_HOST
- Pool size: 5-10 connections
- Timeout: 30 seconds

Full implementation pattern: docs/patterns/database-connection.md
```

### Method 4: Directory References

**For collections of related docs:**
```markdown
## API Documentation

All endpoints documented in: docs/api/

Structure:
- docs/api/authentication.md - Auth endpoints
- docs/api/users.md - User management
- docs/api/products.md - Product catalog
```

Claude can then read specific files as needed.

### Method 5: Conditional References

**Load details only when specific tasks arise:**

```markdown
## Statistical Analysis

When performing hypothesis tests, check:
docs/methodologies/statistical-tests.md

When doing A/B test analysis, reference:
docs/methodologies/ab-testing.md

When building ML models, see:
docs/methodologies/ml-pipeline.md
```

## Complete Working Examples

### Example 1: Multi-Language Project CLAUDE.md

```markdown
# Multi-Language Analytics Platform

## Overview
Analytics platform with Python data engineering and R statistical analysis.

## Quick Reference

**Commands:**
- `make test-all` - Run all tests
- `make setup` - Initialize environments

**Language Details:**
- Python: python/CLAUDE.md
- R: r/CLAUDE.md

## Architecture

Current architecture: Modular monolith with language modules

Full decisions: architecture/decisions/

## Data Flow

Python ETL → data/processed/*.parquet → R analysis → reports/

**Data format specification:** docs/schemas/data-interchange.md

## Cross-Language Conventions

- UTC timestamps
- snake_case variables
- 100 char line limit

**Full style guides:**
- Python: docs/style/python-conventions.md
- R: docs/style/r-conventions.md

## Troubleshooting

Common issues and solutions: docs/troubleshooting/

Most frequent:
- docs/troubleshooting/data-pipeline-errors.md
- docs/troubleshooting/environment-setup.md
```

**How Claude uses this:**
1. Reads inline essentials immediately
2. Loads detailed files only when needed
3. Knows where to find specific information
4. Doesn't waste context on unused details

### Example 2: API Project Documentation Strategy

```markdown
# API Project

## Overview
RESTful API for customer management system

## Quick Start
`npm run dev` - Start development server
`npm test` - Run tests

## API Design Principles

- RESTful resource-oriented
- JWT authentication
- JSON:API response format
- Semantic versioning in path

**Full design guide:** docs/api-design-principles.md

## Standard Response Format

```json
{
  "data": { ... },
  "meta": { "timestamp": "...", "version": "1.0" },
  "errors": []
}
```

**Complete specification:** docs/api/openapi.yaml

## Security Requirements

- Helmet.js for headers
- CORS configured
- Input validation with Joi
- Parameterized queries
- Rate limiting

**Security checklist:** docs/security/checklist.md
**Implementation guide:** docs/security/implementation.md

## Testing

- Jest for unit tests
- Supertest for integration
- Minimum 80% coverage

**Testing patterns:** docs/testing/api-testing-patterns.md

## Deployment

**Deployment procedures:** docs/deployment/

- docs/deployment/staging.md
- docs/deployment/production.md
- docs/deployment/rollback.md
```

### Example 3: Data Science Project

```markdown
# Customer Analytics Pipeline

## Tech Stack
- Python 3.11+
- pandas, scikit-learn
- pytest, black

## Key Commands
- `pytest tests/` - Run tests
- `python -m src.pipeline.run` - Execute pipeline
- `jupyter lab` - Start notebooks

## Data Schemas

**Customers:** data/schemas/customers.md
**Transactions:** data/schemas/transactions.md
**Products:** data/schemas/products.md

Quick reference:
- customer_id: int64, unique
- created_at: datetime64[ns], UTC
- status: ['active', 'inactive', 'churned']

## Statistical Methods

When performing statistical analysis:
docs/methodologies/statistical-guidelines.md

For A/B testing:
docs/methodologies/ab-test-procedure.md

For ML modeling:
docs/methodologies/ml-best-practices.md

## Code Conventions

- Type hints required
- Google-style docstrings
- Use pathlib (not os.path)
- Set random_state=42

**Full Python style guide:** docs/style/python-guide.md

## Pipeline Architecture

```
data/raw/ → ETL → data/processed/ → Analysis → results/
```

**Detailed flow:** docs/architecture/pipeline-architecture.md

## Troubleshooting

Common pipeline errors: docs/troubleshooting/pipeline-errors.md
Data quality issues: docs/troubleshooting/data-quality.md
Environment setup: docs/troubleshooting/environment.md
```

## Best Practices

### DO

**✓ Use plain file paths everywhere:**
```markdown
See docs/api-spec.yaml for endpoints
```

**✓ Keep critical info inline:**
```markdown
## Key Commands
- `make test` - Run tests
- `make deploy` - Deploy to staging
```

**✓ Reference details for deep dives:**
```markdown
Full deployment procedure: docs/deployment/production.md
```

**✓ Organize docs logically:**
```
docs/
├── architecture/    # System design
├── patterns/        # Code patterns
├── api/            # API specs
├── schemas/        # Data schemas
└── troubleshooting/ # Common issues
```

**✓ Use directory references for collections:**
```markdown
All API endpoints: docs/api/
All ADRs: architecture/decisions/
```

**✓ Progressive disclosure:**
```markdown
## Database
Connection pooling enabled (pool_size=5)

Full configuration: docs/database/connection-config.md
```

### DON'T

**✗ Don't use @ syntax in CLAUDE.md:**
```markdown
<!-- WRONG -->
See @docs/api-spec.yaml

<!-- RIGHT -->
See docs/api-spec.yaml
```

**✗ Don't inline everything:**
```markdown
<!-- WRONG: Too much detail -->
## Database Schema
[200 lines of schema details...]

<!-- RIGHT: Summary + reference -->
## Database Schema
Main tables: users, products, orders

Full schema: docs/schemas/database.md
```

**✗ Don't reference files that don't exist:**
```markdown
<!-- Verify files exist first -->
See docs/nonexistent.md  <!-- Claude will error -->
```

**✗ Don't use vague references:**
```markdown
<!-- WRONG -->
See the docs

<!-- RIGHT -->
See docs/api/authentication.md
```

**✗ Don't duplicate content:**
```markdown
<!-- WRONG: Same content in CLAUDE.md and separate file -->

<!-- RIGHT: Essential in CLAUDE.md, details in separate file -->
```

## Documentation Organization Strategy

### Tier 1: CLAUDE.md (Always Loaded)
- Essential project info
- Common commands
- Critical conventions
- References to deeper docs

**Target:** <100 lines

### Tier 2: Referenced Docs (On-Demand)
- Architecture decisions (ADRs)
- API specifications
- Data schemas
- Code patterns
- Troubleshooting guides

**Access:** Via plain file path references

### Tier 3: Deep Reference (Rarely Needed)
- Framework documentation (external links)
- Historical context
- Detailed examples
- Extended tutorials

**Access:** Via links or external URLs

## How to Ask Claude to Read Files

**Direct request:**
```
Read docs/api/authentication.md and implement the JWT flow
```

**Context-building:**
```
Before implementing, read:
1. docs/architecture/decisions/0005-authentication.md
2. docs/patterns/jwt-implementation.md
3. docs/api/authentication.md
```

**Verification:**
```
Check docs/schemas/users.md to verify the user model before proceeding
```

**Troubleshooting:**
```
I'm seeing error X. Check docs/troubleshooting/common-errors.md for solutions
```

## Troubleshooting

### Claude Doesn't Read Referenced Files

**Symptoms:** Claude asks questions answered in referenced docs

**Solutions:**
1. **Make request explicit:** "Read docs/file.md" not just "see docs/file.md"
2. **Verify file path:** Must be relative to project root
3. **Check file exists:** Claude will error on nonexistent files
4. **Be specific about what to find:** "Read docs/api.md and find the authentication section"

### Too Many Files Referenced, Context Bloated

**Symptoms:** Slow responses, context limits hit

**Solutions:**
1. **Don't reference everything upfront:** Only what's needed now
2. **Use progressive disclosure:** Reference details only when needed
3. **Clear context:** Use `/clear` to reset when switching topics
4. **Modularize:** Break large docs into focused files

### References Not Working in Slash Commands

**Symptoms:** Command doesn't access referenced files

**Solutions:**
1. **File paths work, @ syntax doesn't** in command definitions
2. **Instruct Claude explicitly:**
```markdown
Before proceeding, read docs/patterns/test-structure.md
```
3. **Use relative paths** from project root

### CLAUDE.md References Ignored

**Symptoms:** Claude doesn't seem aware of referenced docs

**Solutions:**
1. **Claude doesn't auto-read references** - they're informational
2. **Ask Claude to read when needed:** "Read the API docs referenced in CLAUDE.md"
3. **Consider inline critical info** that's always needed
4. **Use references for details** that are context-dependent

## Advanced Patterns

### Conditional References by Task

```markdown
## Task-Specific Documentation

**For API development:** docs/api/
**For database work:** docs/database/
**For ML modeling:** docs/ml/
**For deployment:** docs/deployment/

Ask Claude to read relevant section when starting work.
```

### Layered Documentation

```markdown
## Documentation Layers

**Level 1 (Always):** This CLAUDE.md file

**Level 2 (Common):**
- architecture/decisions/ - Major decisions
- docs/patterns/ - Code patterns

**Level 3 (Specialized):**
- docs/api/ - API details
- docs/schemas/ - Data schemas
- docs/deployment/ - Deployment procedures

**Level 4 (Reference):**
- External framework docs
- Language documentation
```

### Cross-Referenced Documentation

**In CLAUDE.md:**
```markdown
## API Design
RESTful, versioned endpoints

See: docs/api-design.md
```

**In docs/api-design.md:**
```markdown
# API Design Guidelines

Related documents:
- architecture/decisions/0004-api-versioning.md
- docs/patterns/rest-patterns.md
- docs/security/api-security.md
```

## Next Steps

1. **Organize your docs** into logical directories
2. **Update CLAUDE.md** with plain file path references
3. **Test the pattern:** Ask Claude to read specific files
4. **Iterate:** Add references as you identify needs
5. **Share with team:** Version control doc structure

## Related Guides

- `docs/manuals/claude/how-to/setup-claude-md.md` - CLAUDE.md configuration
- `docs/manuals/claude/how-to/create-slash-command.md` - Reference docs in commands
- `docs/manuals/claude/reference/file-reading.md` - Complete file access reference

## Summary

**Key takeaways:**
- Use plain file paths, not @ syntax
- Keep critical info inline in CLAUDE.md
- Reference detailed docs for deep dives
- Organize docs logically by topic
- Let Claude read files on-demand
- Progressive disclosure keeps context lean
- Explicitly ask Claude to read files when needed

**Remember:** @ syntax is for interactive autocomplete. Plain paths work everywhere and are more explicit.
