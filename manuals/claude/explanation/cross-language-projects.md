# Cross-Language Projects: Polyglot Architecture with AI

## Overview

Modern projects frequently span multiple programming languages: Python for data processing, R for statistical analysis, TypeScript for web interfaces, Swift for mobile apps. Managing these polyglot codebases while maintaining coherence and enabling AI assistance requires intentional architectural decisions. This explanation explores why multi-language projects exist, how to structure them effectively, and how to make them work seamlessly with Claude Code.

## Why Projects Become Multi-Language

### Reason 1: Best Tool for Each Job

Different languages excel at different tasks:

**Python:**
- Machine learning (TensorFlow, PyTorch, scikit-learn ecosystem)
- Data engineering (pandas, polars, extensive library support)
- API development (FastAPI, Flask, Django)
- Scripting and automation

**R:**
- Statistical analysis (built-in statistical functions, comprehensive packages)
- Data visualization (ggplot2, publication-quality graphics)
- Bioinformatics and research (domain-specific packages)
- Reproducible research (R Markdown, Quarto)

**TypeScript/JavaScript:**
- Web frontends (React, Vue, Angular)
- Node.js backends (Express, Nest.js)
- Full-stack development
- Real-time applications

**Rust/Go:**
- Performance-critical components
- System tools
- Concurrent processing

Using the right language for each component yields better results than forcing everything into one language.

### Reason 2: Team Expertise

**Scenario:** Your company has:
- Data scientists proficient in R
- ML engineers expert in Python
- Web developers skilled in TypeScript

**Choice:**
- Option A: Force everyone to use one language (retraining cost, lower productivity)
- Option B: Let teams use their strengths (multi-language codebase)

Most organizations choose Option B, accepting the complexity of polyglot architecture.

### Reason 3: Legacy and Acquisition

**Reality:**
- Acquired a company with different tech stack
- Legacy system in COBOL/Java that can't be rewritten
- Existing tools built in specific languages

**Result:** Organic evolution into multi-language project.

### Reason 4: Platform Requirements

**Example:**
- iOS app requires Swift/Objective-C
- Android app requires Kotlin/Java
- Backend is Python
- Analytics is R
- Web frontend is TypeScript

**Five languages minimum** for this common architecture.

## The Core Challenge: Maintaining Coherence

Multi-language projects face unique challenges:

### Challenge 1: Data Interchange
How does Python talk to R? TypeScript to Python?

**Common pitfalls:**
- CSV files (slow, lose type information, encoding issues)
- JSON (verbose, limited type support, large file sizes)
- Pickle/RDS (language-specific, not portable)

**Better solutions:**
- **Apache Parquet:** Columnar format, preserves types, works across languages
- **Protocol Buffers:** Typed, compact, generated code for multiple languages
- **Apache Arrow:** In-memory columnar format for zero-copy sharing

### Challenge 2: Dependency Management
Each language has its own package manager:
- Python: pip, poetry, conda
- R: CRAN, renv
- JavaScript: npm, yarn, pnpm
- Rust: cargo

**Complexity:** Five languages = five dependency files to maintain.

### Challenge 3: Build Systems
How do you build a project that spans multiple languages?

**Options:**
- Language-specific tools (npm, pip, cargo) called sequentially
- Meta-build systems (Bazel, Nx) that understand multiple languages
- Custom scripts (make, just) orchestrating builds

### Challenge 4: Testing
Each language has different testing frameworks:
- Python: pytest
- R: testthat
- TypeScript: Jest, Vitest
- Rust: built-in testing

**Question:** How do you run "all tests" when "all" means five different frameworks?

### Challenge 5: Documentation
API documentation generated differently per language:
- Python: Sphinx
- R: roxygen2
- TypeScript: JSDoc
- Rust: rustdoc

**Challenge:** Creating unified documentation across languages.

## Architectural Patterns for Polyglot Projects

### Pattern 1: The Modular Monorepo

**Structure:**
```
project/
├── apps/
│   ├── web-frontend/          # TypeScript/React
│   ├── mobile-ios/            # Swift
│   ├── mobile-android/        # Kotlin
│   └── api-gateway/           # Node.js/TypeScript
├── libs/
│   ├── python/
│   │   ├── data-engineering/
│   │   ├── ml-models/
│   │   └── shared-utils/
│   ├── r/
│   │   ├── statistical-analysis/
│   │   └── visualization/
│   └── shared/
│       ├── protobuf-schemas/  # Cross-language contracts
│       └── data-contracts/    # Shared data definitions
├── data/
│   ├── raw/                   # Immutable source data
│   ├── processed/             # Parquet files for interchange
│   └── schemas/               # Schema definitions
├── docs/
│   ├── architecture/
│   └── api/
└── tools/
    └── scripts/               # Build and deployment scripts
```

**Key principles:**
- **Language-specific directories:** Clear boundaries (python/, r/, typescript/)
- **Shared contracts:** Common schemas and interfaces (shared/)
- **Data interchange layer:** Standardized format for language handoffs (data/processed/)
- **Unified documentation:** Central docs/ regardless of implementation language

**Benefits:**
- All code in one repository (atomic changes, unified versioning)
- Clear language boundaries (developers know where to look)
- Explicit interchange points (forces thinking about interfaces)
- Single CI/CD pipeline (simplified deployment)

**Drawbacks:**
- Large repository (slower clones, larger disk usage)
- Build complexity (need orchestration tooling)
- Requires discipline (easy to create spaghetti)

### Pattern 2: Multiple Monorepos by Language

**Structure:**
```
company-python-monorepo/
  apps/
  libs/

company-r-monorepo/
  apps/
  libs/

company-typescript-monorepo/
  apps/
  libs/

data-interchange-repo/
  schemas/
  contracts/
```

**Key principles:**
- Each language has its own monorepo with native tooling
- Separate data-interchange repository defines contracts
- Libraries published as packages (PyPI, CRAN, npm)
- Services communicate via APIs

**Benefits:**
- Language teams have full autonomy (native tooling, conventions)
- Smaller repositories (faster clones, easier navigation)
- Clear ownership (language experts own their repos)
- Independent deployment (services can release separately)

**Drawbacks:**
- Cross-language changes span multiple PRs (coordination overhead)
- Versioning complexity (which Python version works with which R version?)
- Potential for drift (contracts can become inconsistent)

### Pattern 3: Microservices with Polyglot Implementation

**Structure:**
```
service-data-ingestion/        # Python
service-ml-training/           # Python
service-statistical-analysis/  # R
service-api-gateway/          # TypeScript
service-visualization/        # R + Shiny
shared-contracts/             # Protocol Buffers
```

**Key principles:**
- Each service is independent (can be different language)
- Services communicate via defined APIs (REST, gRPC)
- Shared contract repository defines interfaces
- Each service has its own repository and deployment

**Benefits:**
- Maximum autonomy (each service fully independent)
- Technology freedom (choose best language per service)
- Independent scaling (scale services individually)
- Clear APIs (forces good interface design)

**Drawbacks:**
- Distributed system complexity (networking, reliability, observability)
- Debugging difficulty (issues span services)
- Deployment overhead (many services to manage)
- Over-engineering risk (premature microservices)

## Hierarchical CLAUDE.md for Polyglot Projects

The key to making Claude Code effective across languages: **hierarchical context**.

### Root CLAUDE.md: Project-Wide Context

```markdown
# Multi-Language Analytics Platform

## Language Responsibilities
- **Python:** Data engineering, ML model training, API services
- **R:** Statistical analysis, reporting, Shiny dashboards
- **TypeScript:** Web frontend, API gateway

## Data Flow
```
Python ETL → data/processed/*.parquet → R analysis → reports/
                                      ↘ Python ML → models/
```

## Data Interchange Standard
- **Format:** Apache Parquet (preserves types, efficient)
- **Python:** pandas.read_parquet() / df.to_parquet()
- **R:** arrow::read_parquet() / arrow::write_parquet()
- **Schemas:** Defined in shared/schemas/

## Cross-Language Conventions
- Variable names: snake_case in all languages
- Timestamps: UTC timezone, ISO 8601 format
- Column naming: lowercase, underscores, no abbreviations
- Max line length: 100 characters

## Development Commands
- Test all: `make test-all`
- Setup: `make setup`
- Build: `make build`

## Protected Rules
- Never modify data/raw/ (immutable source data)
- Never hardcode credentials (use environment variables)
- Always validate data schemas before processing
```

**Key features:**
- Defines language boundaries and responsibilities
- Specifies data interchange patterns
- Establishes cross-language conventions
- Provides unified commands

### Language-Specific CLAUDE.md Files

**python/CLAUDE.md:**
```markdown
# Python Module Context

## Environment
Python 3.11+, managed with poetry
Activate: `poetry shell`
Install: `poetry install`

## Key Libraries
- pandas, polars for data manipulation
- scikit-learn, torch for ML
- fastapi for APIs
- pytest for testing

## Python Conventions
- Type hints required on all functions
- Google-style docstrings
- Black formatting (100 char lines)
- Imports: standard, third-party, local
- Use pathlib for file paths

## Data Patterns
Load: `df = pd.read_parquet("data/processed/input.parquet")`
Save: `df.to_parquet("data/processed/output.parquet")`
Validate: Use src/utils/schema_validator.py

## Testing
- Tests in tests/ mirror src/ structure
- Fixtures in tests/conftest.py
- Run: `pytest tests/`
- Coverage target: 80%
```

**r/CLAUDE.md:**
```markdown
# R Module Context

## Environment
R 4.3+, managed with renv
Activate: `source("renv/activate.R")`
Restore: `renv::restore()`

## Key Packages
- tidyverse for data manipulation
- arrow for Parquet I/O
- ggplot2 for visualization
- testthat for testing

## R Conventions
- Tidyverse style guide
- Use %>% pipe operator
- roxygen2 documentation
- Snake_case for all names

## Data Patterns
Load: `df <- arrow::read_parquet("data/processed/input.parquet")`
Save: `arrow::write_parquet(df, "data/processed/output.parquet")`
Validate: Use assertr package

## Testing
- Tests in tests/testthat/
- Run: `devtools::test()`
- Use descriptive test names
```

**Benefits of hierarchy:**
When Claude navigates to python/, it loads:
1. Root CLAUDE.md (project-wide context)
2. python/CLAUDE.md (Python-specific conventions)

When Claude navigates to r/, it loads:
1. Root CLAUDE.md (same project context)
2. r/CLAUDE.md (R-specific conventions)

**Result:** Right context at the right time, without overwhelming the base layer.

## Data Interchange: The Critical Integration Point

### The Parquet Pattern

**Why Parquet works across languages:**
- Columnar storage (efficient for analytics)
- Strong typing (preserves int64, float64, datetime, categorical)
- Compression (smaller files, faster I/O)
- Wide support (Python, R, JavaScript, Rust, Java)

**Python implementation:**
```python
import pandas as pd

# Write
df = pd.DataFrame({
    "customer_id": [1, 2, 3],
    "revenue": [100.50, 200.75, 150.25],
    "signup_date": pd.to_datetime(["2024-01-01", "2024-01-02", "2024-01-03"])
})
df.to_parquet("data/processed/customers.parquet", index=False)

# Read
df = pd.read_parquet("data/processed/customers.parquet")
```

**R implementation:**
```r
library(arrow)
library(dplyr)

# Read
df <- read_parquet("data/processed/customers.parquet")

# Process
result <- df %>%
  filter(revenue > 150) %>%
  mutate(revenue_usd = revenue * 1.1)

# Write
write_parquet(result, "data/processed/customers_processed.parquet")
```

**Key observation:** Schema preserved across languages. Python's `int64` becomes R's integer, `datetime` is preserved with timezone.

### Schema Definition and Validation

**shared/schemas/customer.yaml:**
```yaml
name: Customer
description: Customer data schema for cross-language processing
version: "1.0.0"

columns:
  - name: customer_id
    type: int64
    nullable: false
    description: Unique customer identifier

  - name: revenue
    type: float64
    nullable: false
    description: Total customer revenue in USD

  - name: signup_date
    type: datetime64[ns, UTC]
    nullable: false
    description: Customer signup timestamp in UTC

constraints:
  - customer_id > 0
  - revenue >= 0
  - signup_date <= current_timestamp
```

**Python validation:**
```python
import pandera as pa
from pandera import Column, DataFrameSchema

customer_schema = DataFrameSchema({
    "customer_id": Column(int, pa.Check.gt(0)),
    "revenue": Column(float, pa.Check.ge(0)),
    "signup_date": Column("datetime64[ns, UTC]", pa.Check.le(pd.Timestamp.now()))
})

# Validate
validated_df = customer_schema.validate(df)
```

**R validation:**
```r
library(assertr)

df %>%
  verify(customer_id > 0) %>%
  verify(revenue >= 0) %>%
  verify(signup_date <= Sys.time()) %>%
  assert(is.numeric, customer_id, revenue)
```

### API-Based Interchange

For real-time communication between language components:

**Python FastAPI service:**
```python
from fastapi import FastAPI
from pydantic import BaseModel

class PredictionRequest(BaseModel):
    customer_id: int
    features: dict[str, float]

app = FastAPI()

@app.post("/predict")
def predict(request: PredictionRequest):
    prediction = model.predict(request.features)
    return {"customer_id": request.customer_id, "prediction": prediction}
```

**R Shiny consuming API:**
```r
library(httr)
library(jsonlite)

get_prediction <- function(customer_id, features) {
  response <- POST(
    "http://localhost:8000/predict",
    body = list(
      customer_id = customer_id,
      features = features
    ),
    encode = "json"
  )

  content(response, "parsed")
}
```

## Build System Orchestration

### Make-Based Orchestration (Simple Projects)

**Makefile:**
```makefile
.PHONY: setup test build clean

setup:
	cd python && poetry install
	cd r && Rscript -e "renv::restore()"
	npm install

test:
	cd python && poetry run pytest
	cd r && Rscript -e "devtools::test()"
	npm test

build:
	cd python && poetry build
	cd r && R CMD build .
	npm run build

clean:
	cd python && rm -rf dist/ .pytest_cache/
	cd r && rm -rf *.tar.gz
	npm run clean
```

**Usage:**
```bash
make setup  # Install all dependencies
make test   # Run all tests
make build  # Build all components
```

### Nx Monorepo Tooling (Medium Projects)

**nx.json:**
```json
{
  "tasksRunnerOptions": {
    "default": {
      "runner": "nx/tasks-runners/default",
      "options": {
        "cacheableOperations": ["build", "test"]
      }
    }
  }
}
```

**Benefits:**
- Dependency graph understanding (knows what to rebuild)
- Caching (skip unchanged components)
- Parallel execution (run independent tasks simultaneously)
- Affected detection (test only what changed)

### Bazel (Large/Complex Projects)

**WORKSPACE:**
```python
load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

# Python
http_archive(name = "rules_python", ...)

# R (custom rules)
local_repository(name = "rules_r", path = "tools/bazel_rules_r")

# TypeScript
http_archive(name = "build_bazel_rules_nodejs", ...)
```

**Benefits:**
- True polyglot support (Python, R, JS, Rust, Go, etc.)
- Hermetic builds (reproducible across machines)
- Remote caching (share build artifacts across team)
- Scales to Google-size codebases (proven at massive scale)

**Drawbacks:**
- Steep learning curve (complex configuration)
- Bazel-specific build files (BUILD.bazel in every directory)
- Requires investment (only justified at scale)

## Cross-Language Testing Strategies

### Unit Tests: Language-Specific

Each language uses native testing frameworks.

**Python (pytest):**
```python
def test_data_transformation():
    input_df = pd.DataFrame({"value": [1, 2, 3]})
    result = transform_data(input_df)
    assert result["value"].sum() == 6
```

**R (testthat):**
```r
test_that("data transformation works", {
  input_df <- data.frame(value = c(1, 2, 3))
  result <- transform_data(input_df)
  expect_equal(sum(result$value), 6)
})
```

### Integration Tests: Cross-Language

Test data interchange and API contracts.

**Python writes, R reads:**
```python
# Python test
def test_parquet_interchange():
    df = pd.DataFrame({"id": [1, 2, 3], "value": [10, 20, 30]})
    df.to_parquet("test_output.parquet")

    # Call R script to read and process
    subprocess.run(["Rscript", "tests/read_and_validate.R"])

    # Verify R wrote expected output
    result = pd.read_parquet("test_result.parquet")
    assert result["processed"].sum() == 60
```

**R script (tests/read_and_validate.R):**
```r
df <- arrow::read_parquet("test_output.parquet")
result <- df %>% mutate(processed = value * 2)
arrow::write_parquet(result, "test_result.parquet")
```

### Contract Tests: Schema Validation

**shared/tests/test_schemas.py:**
```python
import pandas as pd
import yaml

def test_customer_schema_compliance():
    with open("shared/schemas/customer.yaml") as f:
        schema = yaml.safe_load(f)

    # Python writes conforming data
    df = create_customer_data()
    validate_against_schema(df, schema)

    # R can read and process
    r_result = subprocess.run(
        ["Rscript", "-e", "df <- arrow::read_parquet('test.parquet'); print(nrow(df))"],
        capture_output=True
    )
    assert b"3" in r_result.stdout
```

## AI Context for Polyglot Projects

### CLAUDE.md Strategy

**Root CLAUDE.md:**
- Language responsibilities and boundaries
- Data interchange patterns
- Cross-language conventions
- Unified commands

**Language-specific CLAUDE.md:**
- Language setup and environment
- Language-specific conventions
- Testing and building
- Key libraries and patterns

### Commands for Cross-Language Workflows

**.claude/commands/test-all.md:**
```markdown
---
description: Run complete test suite across all languages
allowed-tools: Bash, Read
---

# Test All Languages

Run comprehensive tests for Python, R, and TypeScript components.

## Execution

1. **Python Tests**
   ```bash
   cd python && poetry run pytest tests/ --cov
   ```

2. **R Tests**
   ```bash
   cd r && Rscript -e "devtools::test()"
   ```

3. **TypeScript Tests**
   ```bash
   npm test
   ```

4. **Integration Tests**
   ```bash
   pytest tests/integration/
   ```

## Report Summary
Display pass/fail for each language and overall coverage.
```

### Subagents for Language Expertise

Consider specialized subagents per language:

**.claude/agents/python-expert.md:**
```markdown
---
name: python-expert
description: Python code expert for data engineering and ML tasks
tools: Read, Write, Edit, Bash
model: sonnet
---

Expert in Python data engineering and ML with pandas, scikit-learn, FastAPI.
Follows project conventions from python/CLAUDE.md.
Specializes in type-hinted, well-tested Python code.
```

**.claude/agents/r-expert.md:**
```markdown
---
name: r-expert
description: R statistical analysis and visualization expert
tools: Read, Write, Edit, Bash
model: sonnet
---

Expert in R statistical analysis with tidyverse, ggplot2, statistical modeling.
Follows project conventions from r/CLAUDE.md.
Specializes in reproducible research and publication-quality visualizations.
```

**Usage:**
- Delegate to python-expert when working in python/ directory
- Delegate to r-expert when working in r/ directory
- Main Claude coordinates between them

## Common Pitfalls and Solutions

### Pitfall 1: Inconsistent Data Types

**Problem:** Python uses int64, R interprets as double, causing type mismatches.

**Solution:**
- Explicit schema definitions in shared/schemas/
- Validation on both sides (write and read)
- Parquet preserves types automatically

### Pitfall 2: Path Handling Differences

**Problem:** Windows vs Unix paths, absolute vs relative references.

**Solution:**
```markdown
# CLAUDE.md Cross-Language Conventions

## Path Handling
- Python: Use pathlib.Path (cross-platform)
- R: Use here::here() package (project-relative)
- Never hardcode absolute paths
- Data paths relative to project root
```

### Pitfall 3: Dependency Version Conflicts

**Problem:** Python needs numpy 1.24, R's reticulate needs numpy 1.23.

**Solution:**
- Document version constraints in root CLAUDE.md
- Use environment isolation (venv, renv)
- Test cross-language interactions in CI

### Pitfall 4: Encoding Issues

**Problem:** R writes UTF-8, Python reads as ASCII.

**Solution:**
```markdown
# CLAUDE.md Standards

## Text Encoding
- All text files: UTF-8
- Python: Explicitly specify encoding="utf-8"
- R: Set options(encoding = "UTF-8")
```

## Decision Framework: When to Go Polyglot

Use multiple languages when:
- [ ] Each language provides significant value (not marginal)
- [ ] Team has expertise in multiple languages
- [ ] Integration points are clear and limited
- [ ] Data interchange is well-defined
- [ ] Build system can handle complexity

Stay single-language when:
- [ ] One language can do everything adequately
- [ ] Team is small (<10 people)
- [ ] Rapid iteration is critical
- [ ] Project is in early stages

## Conclusion

Cross-language projects are powerful but complex. Success requires:

1. **Clear boundaries:** Define which language does what
2. **Standard interchange:** Use Parquet, Protocol Buffers, or APIs
3. **Hierarchical context:** Root + language-specific CLAUDE.md files
4. **Explicit contracts:** Schema definitions, validation, testing
5. **Unified tooling:** Make or Nx or Bazel for orchestration

With intentional architecture and good Claude Code configuration, polyglot projects can leverage the strengths of each language while maintaining coherence and AI-assisted development effectiveness.

The key: Embrace the complexity where it provides value, and constrain it through clear patterns and strong conventions.
