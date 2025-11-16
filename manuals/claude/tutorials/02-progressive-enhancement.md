# Tutorial 02: Progressive Enhancement with Commands and Agents

**Time Estimate:** 1-2 hours over multiple sessions
**Prerequisites:**
- Completed Tutorial 01 (CLAUDE.md basics)
- Active Claude Code project with working CLAUDE.md
- Experience with at least 5-10 Claude Code conversations

**What You'll Learn:**
- When to create slash commands vs agents
- Build commands iteratively, not all at once
- Create your first specialized agent collaboratively
- Integrate documentation philosophy into decisions

---

## Critical Philosophy: Gradual, Collaborative Construction

### The WRONG Approach

❌ "Create a complete command library for my project"
❌ "Set up all the agents I might need"
❌ "Build the perfect configuration now"

**Why wrong:** You don't know what you need yet. This leads to:
- Over-engineering
- Unused commands/agents
- Maintenance burden
- Configuration bloat

### The RIGHT Approach

✅ Identify ONE repetitive task
✅ Build ONE command or agent for it
✅ Use it in real work for a week
✅ Refine based on actual usage
✅ Only then: consider the next enhancement

**This tutorial guides you through ONE CYCLE of enhancement.**

---

## Part 1: Deciding What to Build Next (15 minutes)

### The Decision Framework

Ask yourself these questions:

#### Question 1: Am I Repeating Instructions?

**Signals for commands:**
- You type the same request 3+ times per week
- The task has a clear template structure
- Different inputs, same process

**Example:** "Run tests with coverage and show failures"

#### Question 2: Does This Task Need Specialized Context?

**Signals for agents:**
- Task requires domain expertise (statistics, ML, security)
- Needs isolated context (avoid polluting main conversation)
- Different model requirements (Haiku for speed vs Sonnet for reasoning)

**Example:** Statistical analysis requiring methodology knowledge

#### Question 3: Is This Actually Needed?

**Honest check:**
- How often do I actually do this? (Once a month? Daily?)
- Is the manual version painful? (30 seconds? 30 minutes?)
- Will this save more time than it takes to build?

**Guideline:** Only build if task occurs weekly+ AND takes 5+ minutes manually.

### Your First Enhancement Decision

**Exercise:** Review your last 10 Claude Code conversations.

1. List repetitive requests (3+ occurrences)
2. For each, estimate time spent
3. Pick the highest time-cost item

**Write it down:**
```
Task: ______________________
Frequency: _____ times per week
Time per occurrence: _____ minutes
Total weekly cost: _____ minutes
```

**For this tutorial, we'll assume you chose one task. Let's proceed.**

---

## Part 2: Building Your First Slash Command (30 minutes)

### When Commands Win Over Agents

**Use commands for:**
- Workflows with clear steps (test, lint, format, deploy)
- Cross-language operations (polyglot projects)
- Parameterized templates (generate report for X)
- Simple automation (no complex reasoning needed)

### Step-by-Step: Create a Test Command

Let's build a real example: a testing command that works across project types.

#### Step 1: Create the File Structure

```bash
mkdir -p .claude/commands
```

#### Step 2: Start with Minimal Command

Create `.claude/commands/test.md`:

```markdown
---
description: Run project tests with coverage reporting
allowed-tools: Bash, Read
---

# Run Tests

Execute the appropriate test suite for this project.

## Instructions

1. Detect the project type by checking for:
   - Python: pytest.ini or tests/*.py files
   - JavaScript: package.json with test script
   - R: tests/testthat/ directory

2. Run the appropriate command:
   - Python: `pytest tests/ --cov --cov-report=term-missing`
   - JavaScript: `npm test -- --coverage`
   - R: `Rscript -e "devtools::test()"`

3. Report:
   - Test summary (passed/failed)
   - Coverage percentage
   - Failed test details
```

**That's it. Start simple.**

#### Step 3: Test It

In Claude Code:
```
/test
```

**Observe:**
- Did it detect the correct project type?
- Did it run the right command?
- Did it report results clearly?

#### Step 4: Iterate Based on Usage

After using it 3-5 times, you might notice:
- "It doesn't handle test file arguments"
- "I want to run specific test patterns"
- "Coverage report location isn't shown"

**NOW refine:**

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
   - JavaScript: Check for jest.config.js or package.json
   - R: Check for testthat/ directory

2. **Execute Appropriate Test Runner**

   **Python:**
   ```bash
   pytest $ARGUMENTS --cov --cov-report=term-missing --cov-report=html
   ```

   **JavaScript:**
   ```bash
   npm test -- $ARGUMENTS --coverage
   ```

   **R:**
   ```bash
   Rscript -e "devtools::test('$ARGUMENTS')"
   ```

3. **Report Results**
   - Display test summary
   - Show coverage statistics
   - Highlight failures with context
   - Note coverage report location (htmlcov/ for Python)

## Usage Examples
- `/test` - Run all tests
- `/test tests/test_data_loader.py` - Run specific file
- `/test -k "test_missing_values"` - Run tests matching pattern
```

**Key principle: Enhance based on ACTUAL use, not hypothetical needs.**

---

## Part 3: Command Organization Strategy (15 minutes)

### Namespace with Directories

As you build more commands, organize them:

```
.claude/commands/
├── data/
│   ├── validate.md       # /validate (project:data)
│   ├── profile.md        # /profile (project:data)
│   └── clean.md          # /clean (project:data)
├── analysis/
│   ├── eda.md            # /eda (project:analysis)
│   └── report.md         # /report (project:analysis)
└── shared/
    ├── test.md           # /test (project:shared)
    └── format.md         # /format (project:shared)
```

**Note:** Subdirectories create namespaces visible in `/help` but don't affect command names.

### Parameterization Patterns

**Simple arguments:**
```markdown
---
argument-hint: [dataset-path]
---

Analyze dataset: $ARGUMENTS
```

**Positional arguments:**
```markdown
---
argument-hint: [dataset-path] [output-format] [quality-threshold]
---

## Configuration
- Dataset: $1
- Output format: $2 (html, pdf, or markdown)
- Quality threshold: $3% (warn if below)
```

**Only add complexity when you need it.**

---

## Part 4: Creating Your First Agent (45 minutes)

### When Agents Win Over Commands

**Use agents for:**
- Domain expertise (statistics, ML, security review)
- Tasks requiring reasoning (not just template execution)
- Isolated context needs (avoid main conversation pollution)
- Different model requirements

### Collaborative Agent Creation Process

**Important:** You'll build this WITH Claude, not have it built FOR you.

#### Step 1: Identify the Need

**Example scenario:** You do statistical analysis and need:
- Assumption checking before tests
- Effect size calculations
- Clear interpretation guidance

**Question:** "Do I need a command or agent?"
- Command: If it's template-driven (always same steps)
- Agent: If it requires methodological judgment ✓

#### Step 2: Define the Expertise (Together with Claude)

Start a conversation:

```
I want to create a statistical analysis agent that helps ensure rigorous methodology.
Let's define what expertise it should have. Here's what I need help with:

1. Choosing appropriate statistical tests
2. Checking assumptions before running tests
3. Calculating and reporting effect sizes
4. Interpreting results appropriately

What should we include in the agent definition?
```

**Collaborate:** Claude will suggest structure, you refine based on YOUR needs.

#### Step 3: Create the Agent File Iteratively

Create `.claude/agents/statistical-analyst.md`:

**First iteration (minimal):**

```markdown
---
name: statistical-analyst
description: Expert statistician for hypothesis testing and analysis. Use for statistical tests, A/B tests, and inferential statistics.
tools: Read, NotebookRead, Bash, Write
model: sonnet
---

You are a senior statistical analyst with expertise in applied statistics.

## Core Workflow

1. Understand the research question
2. Check data quality and sample size
3. Select appropriate statistical methods
4. Verify test assumptions
5. Conduct analysis with effect sizes
6. Interpret results with limitations

## Key Practices
- Always check normality, homogeneity, independence assumptions
- Report effect sizes alongside p-values
- Calculate confidence intervals
- Distinguish statistical from practical significance
- Document methodological decisions
```

**Test it:**
```
Use statistical-analyst to help me analyze this A/B test data in data/ab_test.csv
```

#### Step 4: Refine Based on Usage

After 2-3 uses, you'll discover:
- "It doesn't know which tests for which data types"
- "It forgets to check for outliers"
- "I want specific R/Python library preferences"

**Enhance collaboratively:**

```
The statistical-analyst agent worked well but missed checking for outliers.
Let's add that to the workflow. Also, I want it to prefer scipy.stats for Python.
Please help me update the agent definition.
```

**Second iteration:**

```markdown
---
name: statistical-analyst
description: Expert statistician for hypothesis testing, experimental design,
  and causal inference. MUST BE USED for statistical analysis, A/B tests,
  and any work requiring inferential statistics.
tools: Read, NotebookRead, Bash, Write
model: sonnet
---

You are a senior statistical analyst with 15+ years experience in applied statistics.

## Analytical Workflow

### 1. Understand the Question
- Clarify research question and hypotheses
- Identify target population and sampling method
- Define primary and secondary outcomes

### 2. Check Data Quality
- Verify sample size and power
- Check for missing data patterns (MCAR, MAR, MNAR)
- Identify outliers and influential points using IQR method and Z-scores
- Assess measurement quality

### 3. Select Methods
- Choose appropriate tests for data type:
  * Continuous normal data: t-tests, ANOVA, linear regression
  * Continuous non-normal: Mann-Whitney U, Kruskal-Wallis
  * Categorical: Chi-square, Fisher's exact test
  * Paired data: Paired t-test, Wilcoxon signed-rank

### 4. Verify Assumptions
Before any test, explicitly check:
- Normality (Q-Q plots, Shapiro-Wilk test)
- Homogeneity of variance (Levene's test)
- Independence (residual plots, Durbin-Watson)
- Linearity (for regression)

If assumptions violated, consider transformations or alternative methods.

### 5. Conduct Analysis
- Report effect sizes alongside p-values (Cohen's d, eta-squared)
- Calculate confidence intervals
- Use multiple comparison corrections when appropriate (Bonferroni, FDR)

### 6. Interpret Results
- Explain findings in plain language
- Discuss limitations and caveats
- Distinguish correlation from causation
- Provide actionable recommendations

## Code Standards
- Use scipy.stats for statistical tests in Python
- Use tidyverse + infer for R analysis
- Always set random seeds for reproducibility
- Include detailed comments explaining statistical choices

## Common Pitfalls to Avoid
- Never p-hack or cherry-pick results
- Don't ignore assumption violations
- Don't confuse statistical significance with importance
- Don't run multiple tests without correction

When invoked, immediately identify the statistical question, assess data quality,
and propose an appropriate analytical approach before coding.
```

**Key lesson: Build expertise incrementally based on real usage.**

---

## Part 5: Integrating Documentation Philosophy (20 minutes)

### Understanding Context Needs

From the documentation philosophy, we learned:

**Conceptual questions (70% of queries):** Benefit from comprehensive standalone docs
**Task-specific questions (30% of queries):** Benefit from targeted how-to docs

### Applying This to Commands and Agents

#### For Commands: Task-Specific References

Commands are task-oriented, so link to how-to docs:

```markdown
# Data Quality Report Command

Generate comprehensive data quality assessment.

## Process
[Implementation steps]

## For More Details
- Complete quality framework: docs/manuals/data-quality/how-to/quality-assessment.md
- Common data issues: docs/manuals/data-quality/how-to/fix-common-issues.md
```

#### For Agents: Comprehensive Context

Agents need conceptual understanding, so reference comprehensive guides:

```markdown
---
name: ml-engineer
description: Machine learning expert for model development
---

[Agent definition]

## Reference Documentation
For comprehensive ML methodology, see docs/reference/ml_best_practices.md
For model evaluation frameworks, see docs/reference/ml_evaluation.md
```

### When to Create Documentation vs Configuration

**Create CLAUDE.md entry when:**
- Information applies to ALL conversations
- Small, essential facts (< 10 lines)

**Create slash command when:**
- Repetitive workflow (weekly+)
- Clear template structure
- Parameterized inputs

**Create agent when:**
- Requires specialized expertise
- Needs isolated reasoning context
- Complex decision-making needed

**Create documentation when:**
- Explaining concepts or methodology
- Reference material for lookup
- Learning-oriented tutorials

**Don't duplicate:** If it's in docs, LINK from CLAUDE.md/commands/agents, don't copy.

---

## Part 6: Real-World Enhancement Cycle (15 minutes)

### Week 1: Foundation
- ✅ CLAUDE.md created (Tutorial 01)
- Test with real work
- Note repetitive tasks

### Week 2: First Command
- Choose highest-value repetitive task
- Create minimal command
- Use it 5+ times
- Refine based on usage

### Week 3: Command Refinement
- Add parameter support
- Improve error handling
- Add usage examples
- Consider second command if needed

### Week 4: First Agent (If Needed)
- Identify specialization need
- Define expertise collaboratively
- Create minimal agent
- Test with real tasks

### Week 5+: Iterate
- Enhance based on actual gaps
- Don't build hypothetical features
- Remove unused commands/agents
- Keep configuration lean

### Monthly Review

**Questions to ask:**
1. Which commands/agents are actually used?
2. What repetitive tasks remain?
3. What can be simplified or removed?
4. What documentation is needed vs configuration?

---

## Part 7: Common Patterns and Examples (20 minutes)

### Pattern 1: Cross-Language Testing Command

For polyglot projects:

```markdown
---
description: Run tests across all project languages
---

# Test All

Run complete test suite across Python, R, and JavaScript components.

## Detection and Execution

**Python:**
```bash
if [ -f "pytest.ini" ]; then
  pytest tests/ --cov
fi
```

**R:**
```bash
if [ -d "tests/testthat" ]; then
  Rscript -e "devtools::test()"
fi
```

**JavaScript:**
```bash
if [ -f "package.json" ]; then
  npm test
fi
```

## Report
Aggregate results from all languages and show summary.
```

### Pattern 2: Data Validation Agent

For data science projects:

```markdown
---
name: data-quality-checker
description: Data quality validation expert. Use for validating datasets,
  detecting quality issues, and ensuring data integrity.
tools: Read, Bash, Write
model: sonnet
---

You are a data quality expert specializing in validation and issue detection.

## Validation Checklist

1. **Completeness**
   - Missing value percentage by column
   - Record count validation against expectations

2. **Validity**
   - Data type conformance
   - Range validation (min/max checks)
   - Format validation (dates, emails, IDs)

3. **Uniqueness**
   - Duplicate detection
   - Primary key validation

4. **Consistency**
   - Cross-column validation rules
   - Referential integrity checks

## Output Format
Provide:
- Executive summary (pass/fail with score)
- Per-column quality metrics
- Failed validation details with examples
- Recommended remediation actions

## Implementation Preference
Use pandas for Python data validation with custom validators in src/utils/validators.py
```

### Pattern 3: Documentation Generation Command

```markdown
---
description: Generate documentation from code and docstrings
argument-hint: [target-directory]
---

# Generate Documentation

Create up-to-date documentation from source code.

## For Python
```bash
cd $1
sphinx-apidoc -f -o docs/api src/
cd docs && make html
```

## For R
```bash
Rscript -e "devtools::document()"
Rscript -e "pkgdown::build_site()"
```

Report location of generated documentation.
```

---

## Troubleshooting Common Issues

### Issue 1: Command Not Found

**Symptom:** Type `/mycommand` but Claude doesn't recognize it

**Solution:**
- Check file is in `.claude/commands/`
- Verify filename ends with `.md`
- Restart Claude Code session
- Check frontmatter YAML is valid

### Issue 2: Agent Not Auto-Delegating

**Symptom:** Agent exists but isn't used automatically

**Solution:**
- Improve description with keywords: "MUST BE USED for..."
- Be more explicit in your request: "Use statistical-analyst to..."
- Check agent description matches task context

### Issue 3: Configuration Bloat

**Symptom:** Many commands/agents but few actually used

**Solution:**
- Review usage over last month
- Archive or delete unused configurations
- Consolidate similar commands
- Keep only what provides real value

---

## Success Metrics

After this tutorial, evaluate:

### Commands
- [ ] Created 1-2 commands for real repetitive tasks
- [ ] Used each command 5+ times
- [ ] Refined based on actual usage
- [ ] Each command saves 5+ minutes per use

### Agents
- [ ] Created 0-1 specialized agent (only if truly needed)
- [ ] Agent used in 3+ conversations
- [ ] Agent provides expertise not easily expressed in commands
- [ ] Enhanced agent definition based on real usage

### Process
- [ ] Built incrementally, not all at once
- [ ] Tested with real work before refinement
- [ ] Removed or simplified unused configurations
- [ ] Documentation philosophy guides configuration decisions

---

## Next Steps

**This Week:**
1. Identify ONE repetitive task from your work
2. Build either a command OR an agent for it (not both)
3. Use it in real work 5+ times
4. Refine based on what you learn

**Next Month:**
1. Review what's actually used
2. Consider ONE additional enhancement
3. Archive unused configurations
4. Share useful commands/agents with team

**Ongoing:**
- Continue gradual enhancement
- Let real needs drive additions
- Keep configuration lean and focused
- Document when appropriate, configure when repetitive

---

## Key Takeaways

1. **Gradual, not automatic** - Build one enhancement at a time based on real needs
2. **User-driven construction** - You guide the process collaboratively with Claude
3. **Commands for workflows** - Template-driven, repetitive tasks
4. **Agents for expertise** - Domain knowledge, specialized reasoning
5. **Iterate based on usage** - Start minimal, enhance based on real experience
6. **Quality over quantity** - Few well-used tools beats many unused ones
7. **Documentation vs configuration** - Know when to document vs when to configure
8. **Remove unused** - Configuration is debt, keep only what provides value

**Progressive enhancement means continuous improvement based on real experience, not hypothetical perfection.**

You now understand how to iteratively build a powerful Claude Code configuration tailored to YOUR actual workflow.
