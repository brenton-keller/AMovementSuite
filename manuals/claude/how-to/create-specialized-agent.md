# How to Design a Specialized Subagent

## Problem

You have a specific, recurring workflow that:

- Requires deep domain expertise (statistics, ML, code review)
- Benefits from isolated context (doesn't pollute main conversation)
- Should apply consistent best practices every time
- Takes 15+ minutes and involves multiple steps
- Needs to run in parallel with other work

**When NOT to create an agent:**
- You're new to Claude Code (use it for a few weeks first)
- The task is simple (use a slash command instead)
- It only happens once (just ask Claude directly)
- You haven't identified a clear, repeated need

**Golden rule: Create agents AFTER you've manually done the task 5+ times and noticed a consistent pattern.**

## Solution

Create a subagent with specialized system prompts that encode domain expertise and operate in isolated context windows.

## Prerequisites

- 2+ weeks experience using Claude Code
- Clear understanding of the specific task
- Identified 5+ instances where you needed this
- Basic Markdown knowledge
- Understanding of YAML frontmatter

## Step-by-Step Guide

### Step 1: Identify the Need (DON'T SKIP THIS)

Before creating an agent, document:

**What problem does this solve?**
Example: "Statistical analysis tasks require checking assumptions, calculating effect sizes, and documenting methodology - I keep forgetting steps."

**How often does this happen?**
Example: "2-3 times per week for hypothesis testing."

**What expertise does it require?**
Example: "Knowledge of statistical assumptions, appropriate tests, effect size calculations."

**Would a slash command work instead?**
- Simple, stateless tasks → Slash command
- Complex, stateful tasks requiring expertise → Subagent

**Does this need isolated context?**
Example: "Yes, don't want data validation outputs polluting main conversation."

### Step 2: Create the Agent File

```bash
# Project-level (team shared)
mkdir -p .claude/agents
touch .claude/agents/statistical-analyst.md

# Personal (all your projects)
mkdir -p ~/.claude/agents
touch ~/.claude/agents/my-agent.md
```

### Step 3: Basic Agent Structure

**Minimal template:**

```markdown
---
name: agent-name
description: Brief description that helps Claude decide when to delegate
tools: Read, Write, Bash
model: sonnet
---

[System prompt defining expertise and workflow]
```

**Frontmatter fields:**
- `name`: Lowercase-with-hyphens identifier
- `description`: When to use this agent (critical for auto-delegation)
- `tools`: Allowed tools (Read, Write, Edit, Bash, NotebookRead)
- `model`: sonnet, opus, haiku

### Step 4: Write the System Prompt

The system prompt defines the agent's expertise and behavior.

**Template structure:**

```markdown
You are a [role] with [expertise].

## Core Expertise
- [Skill 1]
- [Skill 2]
- [Skill 3]

## Workflow

### Step 1: [Phase Name]
[What to do in this phase]

### Step 2: [Phase Name]
[What to do in this phase]

## Best Practices
- [Practice 1]
- [Practice 2]

## Common Pitfalls to Avoid
- Never [antipattern 1]
- Don't [antipattern 2]

When invoked, [what to do first].
```

### Step 5: Complete Working Example - Statistical Analyst

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

### Step 6: Example - Data Quality Checker

```markdown
---
name: data-quality-checker
description: Validates datasets for completeness, validity, and consistency.
  Use proactively before any analysis or modeling work.
tools: Read, Bash, Write
model: sonnet
---

You are a data quality expert specializing in identifying data issues before
they cause analysis problems.

## Core Expertise
- Missing value detection and pattern analysis
- Data type validation
- Statistical outlier detection
- Referential integrity checking
- Data distribution assessment

## Validation Workflow

### 1. Load and Inspect
- Read dataset and infer schema
- Check row and column counts
- Identify data types
- Generate summary statistics

### 2. Completeness Checks
- Calculate missing value percentage per column
- Identify patterns (MCAR, MAR, MNAR)
- Flag columns with >20% missing
- Check for completely empty columns

### 3. Validity Checks
- Verify data types match expected schema
- Check value ranges (min/max violations)
- Validate formats (dates, emails, phone numbers)
- Check for impossible values (negative ages, future dates)

### 4. Uniqueness Checks
- Detect duplicate rows
- Validate primary key uniqueness
- Check for near-duplicates (fuzzy matching)

### 5. Consistency Checks
- Cross-column validation rules
- Referential integrity with related tables
- Business rule violations
- Temporal consistency (start < end dates)

### 6. Distribution Analysis
- Generate histograms for numerical columns
- Identify skewness and outliers (IQR, Z-scores)
- Check categorical value distributions
- Flag unexpected cardinality

### 7. Generate Report
Create summary with:
- Executive summary (pass/fail with score)
- Per-column quality metrics
- Failed validation details with examples
- Recommended remediation actions
- Data quality score (0-100)

## Quality Scoring
- 90-100: Excellent (minor issues only)
- 80-89: Good (some cleaning needed)
- 70-79: Fair (significant issues)
- <70: Poor (major data problems)

## Best Practices
- Always visualize distributions before statistical tests
- Document all assumptions about expected data
- Provide specific examples of violations
- Suggest concrete remediation steps
- Distinguish data errors from valid outliers
- Generate reproducible validation scripts

## Output Format
Save report to: reports/quality/[dataset-name]-quality-report.md

Include:
1. Executive Summary
2. Completeness Assessment
3. Validity Assessment
4. Uniqueness Assessment
5. Consistency Assessment
6. Distribution Analysis
7. Recommendations
8. Validation Code (for reproducibility)

When invoked, ask for dataset path and expected schema if not provided,
then proceed with comprehensive validation.
```

### Step 7: Example - Code Reviewer

```markdown
---
name: code-reviewer
description: Reviews code for quality, best practices, security, and
  maintainability. Use for all pull requests and significant code changes.
tools: Read
model: sonnet
---

You are a senior software engineer and code reviewer with expertise across
multiple languages and best practices.

## Review Focus Areas

### 1. Correctness
- Logic errors and edge cases
- Proper error handling
- Boundary conditions
- Off-by-one errors

### 2. Code Quality
- Readability and clarity
- DRY principle adherence
- Single Responsibility Principle
- Appropriate abstraction levels

### 3. Testing
- Test coverage for new code
- Edge cases tested
- Integration test needs
- Test quality and clarity

### 4. Security
- Input validation
- SQL injection prevention
- XSS vulnerabilities
- Authentication/authorization
- Secrets management

### 5. Performance
- Algorithmic complexity
- Database query efficiency
- Memory usage
- Caching opportunities

### 6. Maintainability
- Documentation quality
- Code organization
- Naming conventions
- Comment clarity

## Review Process

### 1. Understand Context
- Read related code
- Check existing patterns
- Review linked issues/tickets
- Understand intent

### 2. Analyze Changes
- Review diff thoroughly
- Check for unintended changes
- Verify tests updated
- Check documentation

### 3. Provide Feedback
Structure comments as:
- **Critical:** Must fix (security, correctness)
- **Important:** Should fix (quality, maintainability)
- **Suggestion:** Nice to have (style, optimization)
- **Question:** Clarification needed
- **Praise:** What's done well

### 4. Generate Summary
- Overall assessment (approve, request changes, comment)
- Count of critical/important/suggestion items
- Key strengths
- Key concerns
- Estimated effort to address

## Best Practices
- Be specific with examples
- Suggest alternatives, don't just criticize
- Explain the "why" behind suggestions
- Recognize good patterns
- Consider project context
- Balance nitpicking with pragmatism

## Common Issues to Flag
- Missing error handling
- Hardcoded credentials or config
- Commented-out code
- Console.log/print statements
- Inconsistent naming
- Missing tests for new features
- Breaking API changes without migration plan
- Database queries in loops (N+1)

## Tone
- Professional and constructive
- Focus on code, not person
- Assume competence and good intent
- Ask questions rather than demand
- Celebrate improvements

When invoked, read the code changes thoroughly, understand context,
then provide structured, actionable feedback.
```

### Step 8: Test Your Agent

**Explicit invocation:**
```
Use [agent-name] to [task]
```

Example:
```
Use statistical-analyst to analyze the A/B test results in data/experiment-results.csv
```

**Automatic delegation (requires good description):**
```
Analyze the hypothesis test results and check all assumptions
```

If description is well-written, Claude will automatically delegate to the right agent.

### Step 9: Refine Based on Usage

After 5-10 uses:

1. **Check delegation accuracy:**
   - Does Claude auto-delegate when appropriate?
   - Improve `description` if not

2. **Review outputs:**
   - Are steps followed consistently?
   - Add missing best practices
   - Remove unnecessary steps

3. **Update expertise:**
   - Add new domain knowledge learned
   - Refine workflow based on real usage

## Advanced Patterns

### Different Models for Different Agents

```markdown
---
name: quick-validator
model: haiku  # Fast and cheap for simple validation
---
```

```markdown
---
name: complex-analyst
model: opus  # More powerful for complex reasoning
---
```

### Coordinated Multi-Agent Workflows

**Main conversation orchestrates:**
1. data-quality-checker validates data
2. statistical-analyst performs analysis
3. visualization-specialist creates charts
4. code-reviewer reviews generated code

Each agent works in isolation, returns results to main conversation.

## Troubleshooting

### Agent Not Being Auto-Delegated

**Cause:** Description doesn't match task clearly

**Solutions:**
1. Improve description with keywords: "MUST BE USED for [scenarios]"
2. Add specific triggers: "Use for hypothesis testing, A/B tests"
3. Explicitly invoke: "Use [agent-name] to..."
4. Check competing agent descriptions

### Agent Has Wrong Context

**Cause:** Tools not allowed in frontmatter

**Solutions:**
1. Add required tools: `tools: Read, Write, Bash, NotebookRead`
2. Each agent has isolated context (by design)
3. Pass necessary info explicitly when invoking

### Agent Doesn't Follow Workflow

**Cause:** Instructions unclear or too vague

**Solutions:**
1. Make steps more explicit
2. Add numbered workflows
3. Include examples of expected behavior
4. Add "When invoked, [first action]" at end

### Too Many Agents, Getting Confused

**Cause:** Over-specialization

**Solutions:**
1. Consolidate similar agents
2. Start with 2-3 agents, expand only when clear need
3. Name descriptively: `statistical-analyst` not `stats-agent`
4. Maintain agent inventory document

## Best Practices

### DO

- ✓ **Wait to create:** Use Claude Code for weeks first
- ✓ **Identify real needs:** 5+ manual instances of same task
- ✓ **Start with 2-3 agents:** Don't create a zoo
- ✓ **Encode domain expertise:** Best practices, common pitfalls
- ✓ **Write clear descriptions:** Aids auto-delegation
- ✓ **Test explicitly first:** Before relying on auto-delegation
- ✓ **Iterate based on usage:** Refine after real-world use
- ✓ **Document workflows:** Numbered steps, clear phases

### DON'T

- ✗ **Create on day 1:** You don't know your needs yet
- ✗ **Make agents for one-offs:** Use slash commands or just ask
- ✗ **Over-specialize:** Don't need 20 agents
- ✗ **Forget about isolation:** Context is separate (by design)
- ✗ **Write vague prompts:** Be explicit about expertise
- ✗ **Skip testing:** Verify behavior before relying on it
- ✗ **Set and forget:** Update based on learnings

## Decision Matrix: Command vs. Agent

| Factor | Use Slash Command | Use Subagent |
|--------|------------------|--------------|
| **Complexity** | Simple, 1-5 steps | Complex, 5+ steps |
| **State** | Stateless | May need context |
| **Expertise** | Basic instructions | Domain expertise required |
| **Context** | Can share context | Benefits from isolation |
| **Frequency** | Very frequent (daily) | Regular (weekly) |
| **Time** | <5 minutes | 15+ minutes |
| **Examples** | Format code, run tests | Statistical analysis, code review |

## Agent Development Timeline

**Week 1-2 of Claude Code:**
- ❌ Don't create any agents yet
- ✓ Use Claude Code naturally
- ✓ Notice repeated patterns

**Week 3-4:**
- ✓ Identify 1-2 clear needs
- ✓ Create first agent
- ✓ Test explicitly

**Month 2-3:**
- ✓ Add 1-2 more agents if needed
- ✓ Refine existing agents
- ✓ Share with team

**Month 4+:**
- ✓ Mature agent library (3-5 agents)
- ✓ Optimize descriptions for auto-delegation
- ✓ Document agent usage patterns

## Next Steps

1. **Wait** (if you're new to Claude Code - use it for 2+ weeks first)
2. **Identify** a clear, repeated need (5+ instances)
3. **Create** one agent using the templates above
4. **Test** explicitly before relying on auto-delegation
5. **Refine** based on real usage
6. **Expand** slowly and deliberately

## Related Guides

- `docs/manuals/claude/how-to/create-slash-command.md` - For simpler, stateless tasks
- `docs/manuals/claude/how-to/setup-claude-md.md` - Project-wide context
- `docs/manuals/claude/reference/subagents.md` - Complete subagent reference

## Additional Resources

- Official Anthropic subagent documentation
- Pre-built agents: github.com/VoltAgent/awesome-claude-code-subagents
- Agent templates: aitmpl.com
- Community patterns: github.com/hesreallyhim/awesome-claude-code
