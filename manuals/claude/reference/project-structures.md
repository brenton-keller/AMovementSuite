# Project Structures Reference

Recommended directory structures by project type for Claude Code optimization.

## Overview

Well-organized projects make Claude Code dramatically more productive by creating clear boundaries and explicit structure. This reference provides battle-tested patterns for different project types.

## Structure Selection Matrix

| Project Type | Team Size | Structure | Complexity |
|--------------|-----------|-----------|------------|
| Single-language prototype | 1-3 | Flat feature-based | Low |
| Single-language MVP | 3-10 | Modular monolith | Medium |
| Multi-language data science | 2-20 | Language-separated monorepo | Medium |
| API service | 5-20 | Layered architecture | Medium |
| Full-stack application | 10-50 | Apps + libs monorepo | High |
| Enterprise platform | 50+ | Hybrid polyrepo | Very high |

## Base Patterns

### Pattern 1: Modular Monolith (Default)

**When to use:**
- Starting new projects (Google research recommends as default)
- Teams of 2-50 developers
- Need rapid iteration without distributed system complexity

**Structure:**
```
project/
├── .claude/
│   ├── CLAUDE.md              # Project context
│   ├── commands/              # Slash commands
│   │   ├── data/              # Data processing commands
│   │   ├── ml/                # ML workflow commands
│   │   └── shared/            # Cross-domain commands
│   ├── agents/                # Specialized subagents
│   │   ├── data-engineer.md
│   │   ├── ml-engineer.md
│   │   └── code-reviewer.md
│   └── settings.json          # Project settings
├── src/
│   └── modules/               # Feature modules
│       ├── user-management/
│       │   ├── domain/        # Business logic
│       │   ├── application/   # Use cases
│       │   ├── infrastructure/# External concerns
│       │   └── api/           # Public interface
│       ├── payment-processing/
│       └── analytics/
├── shared-kernel/             # Shared domain concepts
│   ├── domain/
│   └── infrastructure/
├── docs/
│   ├── architecture/
│   │   └── adr/              # Architecture Decision Records
│   ├── api/
│   └── development/
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── tools/                     # Development scripts
└── README.md
```

**CLAUDE.md example:**
```markdown
# Project Name

## Architecture
Modular monolith with feature-based organization

## Module Structure
Each module in src/modules/ follows:
- domain/ - Pure business logic
- application/ - Use cases
- infrastructure/ - External integrations
- api/ - Public interfaces

## Commands
- `npm run build`
- `npm test`
- `npm run dev`

## Module Boundaries
Modules communicate through public APIs only.
No direct imports from other module internals.

## Protected
- ❌ No circular dependencies between modules
- ❌ Don't bypass public APIs
```

### Pattern 2: Multi-Language Monorepo

**When to use:**
- Polyglot projects (Python + R + JavaScript, etc.)
- Heavy code sharing needed
- Atomic changes across languages critical

**Structure:**
```
monorepo/
├── .claude/
│   ├── CLAUDE.md              # Root context
│   ├── commands/
│   │   ├── shared/            # Cross-language commands
│   │   ├── python/            # Python-specific
│   │   └── r/                 # R-specific
│   └── agents/
├── apps/                      # Deployable applications
│   ├── web-frontend/          # TypeScript/React
│   ├── mobile-ios/            # Swift
│   ├── api-gateway/           # Node.js
│   ├── ml-pipeline/           # Python
│   └── data-analysis/         # R
├── libs/                      # Shared libraries
│   ├── typescript/
│   │   ├── ui-components/
│   │   └── api-client/
│   ├── python/
│   │   ├── data-utils/
│   │   └── ml-models/
│   └── shared/                # Cross-language (Protobuf, etc.)
├── docs/
│   ├── architecture/
│   └── languages/             # Language-specific guides
└── tools/                     # Build and dev tools
```

**Root CLAUDE.md:**
```markdown
# Multi-Language Platform

## Languages
- TypeScript/JavaScript: Frontend, API gateway
- Python: ML, data engineering
- R: Statistical analysis, reporting
- Swift: iOS mobile app

## Data Interchange
Use Parquet for cross-language data exchange.
Never use CSV for large datasets.

## Conventions (all languages)
- Snake_case for variable names
- UTC timezone for timestamps
- Maximum line length: 100 characters

## Build
Tool: Nx for TypeScript/JavaScript, Bazel for polyglot
Commands: `nx build [app]`, `bazel build //...`

## Testing
- `nx test [project]` - JS/TS tests
- `pytest` - Python tests
- `devtools::test()` - R tests
```

### Pattern 3: Data Science Project

**When to use:**
- Research-heavy workflows
- Notebook-based development
- Statistical analysis and ML

**Structure:**
```
data-science-project/
├── .claude/
│   ├── CLAUDE.md
│   ├── commands/
│   │   ├── data/
│   │   │   ├── validate.md
│   │   │   ├── clean.md
│   │   │   └── profile.md
│   │   ├── analysis/
│   │   │   ├── eda.md
│   │   │   └── hypothesis.md
│   │   └── ml/
│   │       ├── train.md
│   │       ├── evaluate.md
│   │       └── deploy.md
│   └── agents/
│       ├── data-quality-checker.md
│       ├── statistical-analyst.md
│       ├── ml-engineer.md
│       └── visualization-specialist.md
├── data/
│   ├── raw/                   # Original, immutable data
│   ├── processed/             # Cleaned, transformed data
│   └── schemas/               # Data schema definitions
├── notebooks/                 # Jupyter/R notebooks
│   ├── exploratory/
│   ├── analysis/
│   └── reports/
├── src/                       # Production code
│   ├── data/                  # Data loading, validation
│   ├── features/              # Feature engineering
│   ├── models/                # Model training, evaluation
│   └── visualization/         # Plotting utilities
├── models/                    # Trained model artifacts
├── reports/                   # Generated reports
├── tests/
├── docs/
│   ├── data-dictionary.md
│   ├── methodology.md
│   └── experiments/           # Experiment logs
└── README.md
```

**CLAUDE.md:**
```markdown
# Data Science Project

## Stack
Python 3.11, pandas, scikit-learn, jupyter, pytest

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
- `jupyter lab` - Start notebook server
- `pytest tests/` - Run tests
- `python -m src.pipeline.run` - Execute pipeline

## Data Flow
raw/ → processing → processed/ → analysis → reports/

## Protected
- ❌ Never modify data/raw/
- ❌ Always set random seed before stochastic operations
- ✅ Save processed data to data/processed/
```

### Pattern 4: API Service

**When to use:**
- REST or GraphQL APIs
- Microservice or service-oriented
- Backend services

**Structure:**
```
api-project/
├── .claude/
│   ├── CLAUDE.md
│   ├── commands/
│   │   ├── test.md
│   │   └── deploy.md
│   └── agents/
│       └── code-reviewer.md
├── src/
│   ├── routes/                # API endpoints
│   │   ├── auth.ts
│   │   ├── users.ts
│   │   ├── products.ts
│   │   └── orders.ts
│   ├── models/                # Data models
│   ├── services/              # Business logic
│   ├── middleware/            # Auth, validation, errors
│   │   ├── auth.ts
│   │   ├── validation.ts
│   │   └── error-handler.ts
│   ├── utils/
│   └── server.ts
├── tests/
│   ├── integration/           # Full request/response tests
│   └── unit/                  # Service/utility tests
├── docs/
│   ├── api-spec.yaml          # OpenAPI specification
│   └── architecture/
├── config/
└── README.md
```

**CLAUDE.md:**
```markdown
# API Service

## Stack
Node.js 20, Express, PostgreSQL, Jest

## API Design
- RESTful resource-oriented
- JWT authentication (all endpoints except /auth/login)
- JSON:API response format
- Rate limit: 100 requests/minute per user

## Standard Response
```json
{
  "data": { ... },
  "meta": { "timestamp": "...", "version": "1.0" },
  "errors": []
}
```

## Error Codes
- 400: Bad request (validation failed)
- 401: Unauthorized (invalid/missing token)
- 403: Forbidden (insufficient permissions)
- 404: Not found
- 429: Too many requests (rate limit)
- 500: Internal error (never expose details)

## Security
- Helmet.js for security headers
- CORS properly configured
- Input validation with Joi schemas
- Parameterized queries (prevent SQL injection)
- Secrets in environment variables only

## Testing
- Jest for unit tests
- Supertest for integration tests
- Minimum 80% coverage
- Mock external dependencies

## Commands
- `npm test`
- `npm run dev`
- `npm run build`

## Protected
- ❌ No credentials in code
- ❌ No raw SQL strings (use parameterized queries)
- ✅ Validate all inputs
```

### Pattern 5: Dual-Track (Research + Production)

**When to use:**
- Experimental features alongside stable code
- Research-heavy projects
- Need clear graduation path

**Structure:**
```
project/
├── .claude/
│   ├── CLAUDE.md
│   ├── commands/
│   └── agents/
├── production/                # Stable, deployed code
│   ├── services/
│   ├── libs/
│   └── deployments/
├── research/                  # Experimental work
│   ├── experiments/
│   │   ├── feature-x-spike/
│   │   │   ├── hypothesis.md
│   │   │   ├── setup.md
│   │   │   ├── results.md
│   │   │   └── code/
│   │   └── performance-study/
│   ├── prototypes/            # Working prototypes
│   └── analysis/              # Research outputs
├── staging/                   # Graduation path
│   └── feature-y/             # Moving toward production
├── docs/
│   ├── decisions/             # ADRs documenting promotion
│   └── experiments/           # Experiment logs
└── README.md
```

**Graduation checklist template:**
```markdown
# docs/decisions/feature-x-promotion.md

## Promotion Checklist: Feature X

### Research → Staging
- [ ] Hypothesis validated with data
- [ ] Technical feasibility proven
- [ ] Performance acceptable (<100ms latency)
- [ ] Security review completed
- [ ] Architecture Decision Record written

### Staging → Production
- [ ] Unit test coverage >80%
- [ ] Integration tests passing
- [ ] Documentation complete
- [ ] Staged rollout plan defined
- [ ] Rollback procedure documented
- [ ] Monitoring/alerting configured
```

**CLAUDE.md:**
```markdown
# Dual-Track Project

## Structure
- production/ - Stable code (deployed)
- research/ - Experiments and prototypes
- staging/ - Features being promoted

## Promotion Process
Research → Staging → Production
See docs/decisions/ for ADRs documenting each promotion.

## Research Guidelines
- Document hypothesis in hypothesis.md
- Track results in results.md
- Keep code self-contained
- No dependencies on production code

## Production Standards
- 80%+ test coverage
- Full documentation
- Monitoring configured
- Rollback plan documented

## Commands
- Production: `npm run prod:build`, `npm run prod:deploy`
- Research: `npm run research:test`
```

## Language-Specific Structures

### Python Package

```
python-package/
├── .claude/
│   └── CLAUDE.md
├── src/
│   └── package_name/
│       ├── __init__.py
│       ├── module1.py
│       └── module2.py
├── tests/
│   ├── conftest.py
│   └── test_module1.py
├── docs/
├── pyproject.toml
├── README.md
└── LICENSE
```

**CLAUDE.md:**
```markdown
# Python Package

## Environment
Python 3.11+, Poetry for dependencies

## Setup
- `poetry install`
- `poetry shell`

## Code Standards
- Type hints required
- Google-style docstrings
- Black formatting
- Ruff linting
- Minimum 80% test coverage

## Commands
- `poetry run pytest`
- `poetry run black .`
- `poetry run ruff check .`
- `poetry build`
```

### R Package

```
r-package/
├── .claude/
│   └── CLAUDE.md
├── R/                         # Function definitions
├── man/                       # Auto-generated docs (roxygen2)
├── tests/
│   └── testthat/
├── vignettes/                 # Long-form documentation
├── data/                      # Example datasets
├── data-raw/                  # Scripts creating data/
├── DESCRIPTION
├── NAMESPACE
└── README.md
```

**CLAUDE.md:**
```markdown
# R Package

## Environment
R 4.3+, renv for dependencies

## Setup
- `renv::restore()`

## Development Commands
- `devtools::load_all()` - Load package
- `devtools::document()` - Generate docs from roxygen
- `devtools::test()` - Run tests
- `devtools::check()` - Full validation
- `devtools::install()` - Install locally

## Code Standards
- Follow tidyverse style guide
- Use roxygen2 for documentation
- Export functions explicitly with @export
- Document all parameters with @param

## Protected
- ❌ Never modify man/ directly (generated from roxygen)
- ❌ Don't change exported function signatures (breaking)
```

### R Shiny Application

```
shiny-app/
├── .claude/
│   └── CLAUDE.md
├── R/
│   ├── modules/               # Shiny modules
│   │   ├── data_module.R
│   │   ├── plot_module.R
│   │   └── stats_module.R
│   ├── utils/                 # Helper functions
│   └── global.R               # Global setup
├── www/                       # Static assets
│   ├── css/
│   └── js/
├── data/                      # App data
├── tests/
│   └── testthat/
├── app.R                      # App entry point
└── README.md
```

**CLAUDE.md:**
```markdown
# Shiny Application

## Architecture
Modular structure with bslib for UI

## Module Pattern
All modules follow:
- `module_ui(id)` - UI function with NS()
- `module_server(id)` - Server function with moduleServer()
- Always namespace IDs: `ns <- NS(id)`

## Reactive Structure
- Use reactive() for data processing
- Use observeEvent() for side effects
- Use renderPlot()/renderTable() for outputs
- Isolate expensive operations with reactiveVal()

## Best Practices
- Validate inputs before processing
- Use req() for required inputs
- Provide loading indicators
- Handle errors gracefully
- Test reactivity thoroughly

## Commands
- `shiny::runApp()`
- `devtools::test()`

## Common Pitfalls
- ❌ Don't use reactive values outside reactive contexts
- ❌ Don't create infinite reactive loops
- ❌ Don't forget to namespace IDs in modules
```

## Tooling-Specific Structures

### Nx Monorepo (TypeScript/JavaScript)

```
nx-workspace/
├── .claude/
│   └── CLAUDE.md
├── apps/
│   ├── web/
│   ├── mobile/
│   └── api/
├── libs/
│   ├── shared/
│   │   ├── ui/
│   │   ├── data-access/
│   │   └── utils/
│   └── feature/
├── tools/
├── nx.json
├── workspace.json
└── package.json
```

**CLAUDE.md:**
```markdown
# Nx Workspace

## Structure
- apps/ - Deployable applications
- libs/ - Shared libraries

## Module Boundaries
Enforced by nx.json configuration.
Each lib has public API in index.ts.

## Commands
- `nx build [app]`
- `nx test [project]`
- `nx affected:test` - Test affected by changes
- `nx graph` - View dependency graph

## Conventions
- No circular dependencies
- Libs depend on libs, apps depend on libs
- Shared types in libs/shared/types
```

### Bazel Polyglot

```
bazel-workspace/
├── .claude/
│   └── CLAUDE.md
├── python/
│   ├── BUILD.bazel
│   └── src/
├── go/
│   ├── BUILD.bazel
│   └── src/
├── typescript/
│   ├── BUILD.bazel
│   └── src/
├── shared/
│   └── proto/                 # Protocol buffers
│       └── BUILD.bazel
├── WORKSPACE
└── BUILD.bazel
```

**CLAUDE.md:**
```markdown
# Bazel Polyglot Workspace

## Languages
- Python 3.11
- Go 1.21
- TypeScript 5.0

## Build
Tool: Bazel with remote caching

## Commands
- `bazel build //...` - Build all
- `bazel test //...` - Test all
- `bazel build //python/...` - Build Python only
- `bazel run //:target` - Run target

## Shared Interfaces
Protocol Buffers in shared/proto/
```

## Migration Paths

### Flat Script → Modular Monolith

**Before:**
```
script-project/
├── main.py               # Everything in one file
├── data.csv
└── requirements.txt
```

**After (Week 2):**
```
script-project/
├── .claude/
│   └── CLAUDE.md
├── src/
│   ├── data/
│   │   ├── __init__.py
│   │   └── loader.py
│   ├── processing/
│   │   ├── __init__.py
│   │   └── processor.py
│   └── analysis/
│       ├── __init__.py
│       └── analyzer.py
├── tests/
├── data/
│   ├── raw/
│   └── processed/
├── main.py               # Orchestrates modules
└── requirements.txt
```

### Monolith → Microservices

Extract module when:
- [ ] Independent scaling needed
- [ ] Team ownership clear
- [ ] API stable for 6+ months
- [ ] Benefits justify distribution costs

**Strangler Fig Pattern:**
1. Identify module to extract
2. Create API adapter layer
3. Deploy module as separate service
4. Route traffic through adapter
5. Remove from monolith when stable

## Best Practices Summary

### Directory Naming

| Good | Bad | Why |
|------|-----|-----|
| `src/modules/user-management/` | `src/um/` | Clear, descriptive |
| `tests/integration/` | `tests/int/` | No abbreviations |
| `docs/architecture/` | `docs/arch/` | Full words |
| `.claude/commands/data/` | `.claude/cmds/data/` | Explicit |

### File Organization

**Principles:**
1. **Group by feature** (not by type)
2. **Clear boundaries** (explicit interfaces)
3. **Minimal coupling** (independent modules)
4. **Progressive disclosure** (start flat, add structure as needed)

**Wrong (group by type):**
```
src/
├── controllers/
│   ├── user.ts
│   └── product.ts
├── models/
│   ├── user.ts
│   └── product.ts
└── services/
    ├── user.ts
    └── product.ts
```

**Right (group by feature):**
```
src/modules/
├── user-management/
│   ├── controller.ts
│   ├── model.ts
│   └── service.ts
└── product-catalog/
    ├── controller.ts
    ├── model.ts
    └── service.ts
```

### Protected Areas

Mark clearly in CLAUDE.md:
```markdown
## Protected Areas
- ❌ Never modify data/raw/ (immutable source)
- ❌ Don't bypass module APIs (use public interfaces)
- ❌ No circular dependencies (enforce with tooling)
- ❌ Don't hardcode credentials (use env vars)
```

## Validation Checklist

- [ ] Structure matches project type and team size
- [ ] Clear `.claude/` organization
- [ ] CLAUDE.md documents structure
- [ ] Protected areas marked
- [ ] Module boundaries explicit
- [ ] Testing structure mirrors source
- [ ] Documentation in docs/
- [ ] Build/dev tools in tools/
- [ ] Version control configured (.gitignore)

## See Also

- **CLAUDE.md Specification**: claude-md-specification.md
- **Command Specification**: command-specification.md
- **Agent Specification**: agent-specification.md
- **File Path Syntax**: file-path-syntax.md
