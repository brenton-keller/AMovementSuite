# Claude Code Manual

**Transform your development workflow through intelligent context management, specialized agents, and reusable commands.**

## Quick Navigation

### I'm new to Claude Code...
Start here to build foundational understanding:
1. **[Getting Started](tutorials/01-getting-started.md)** (30-60 min) - First steps, basic CLAUDE.md setup
2. **[Progressive Enhancement](tutorials/02-progressive-enhancement.md)** (1-2 hours) - Gradually add commands and agents
3. **[Documentation Philosophy](explanation/documentation-philosophy.md)** - Understand AI-first vs human-first documentation

### I need to accomplish something specific...
Task-oriented guides for common workflows:
- **[Setup CLAUDE.md](how-to/setup-claude-md.md)** - Configure project context effectively
- **[Create a Slash Command](how-to/create-slash-command.md)** - Automate repetitive tasks
- **[Create a Specialized Agent](how-to/create-specialized-agent.md)** - Build domain expertise (after 2+ weeks of use)
- **[Cross-Reference Documentation](how-to/cross-reference-docs.md)** - Reference files without @ syntax

### I'm looking up specific information...
Comprehensive reference documentation:
- **[CLAUDE.md Specification](reference/claude-md-specification.md)** - Complete syntax and options
- **[Command Specification](reference/command-specification.md)** - Slash command YAML and parameters
- **[Agent Specification](reference/agent-specification.md)** - Subagent definition format
- **[Project Structures](reference/project-structures.md)** - Recommended directory layouts
- **[File Path Syntax](reference/file-path-syntax.md)** - How to reference files (NOT @ syntax!)

### I want to understand the "why" behind concepts...
Deep dives into architecture and design decisions:
- **[Configuration Layers](explanation/configuration-layers.md)** - Why CLAUDE.md, commands, and agents
- **[Iterative Enhancement](explanation/iterative-enhancement.md)** - Why NOT to build complex setups immediately
- **[Cross-Language Projects](explanation/cross-language-projects.md)** - Multi-language architecture
- **[Modular Monolith Pattern](explanation/modular-monolith-pattern.md)** - Why this works for AI-augmented projects

---

## Complete Manual Contents

### Tutorials (Learning-Oriented)

**[01 - Getting Started](tutorials/01-getting-started.md)** (30-60 min)
Setting up Claude Code from scratch. Learn the three-layer configuration system (CLAUDE.md, commands, agents), write your first project context, and understand version control integration. Perfect for absolute beginners.

**[02 - Progressive Enhancement](tutorials/02-progressive-enhancement.md)** (1-2 hours)
Build your Claude Code configuration iteratively over weeks, not hours. Learn when to create commands vs agents, how to organize by real needs (not hypothetical ones), and follow a week-by-week enhancement cycle. Emphasizes collaborative, gradual construction.

### How-To Guides (Task-Oriented)

**[Create a Slash Command](how-to/create-slash-command.md)**
Automate repetitive workflows with custom commands. Learn the 6-step creation process, multi-language detection patterns, parameterization, and troubleshooting. Includes complete examples for test runners, formatters, and data quality reports.

**[Setup CLAUDE.md](how-to/setup-claude-md.md)**
Configure effective project context that loads automatically. Learn what to include (and exclude), how to stay under 100 lines, import system usage, and hierarchical patterns for multi-language projects. Includes templates for data science, API, and R package projects.

**[Create a Specialized Agent](how-to/create-specialized-agent.md)**
Build domain-specific subagents for complex workflows. **Important:** Only create agents after 2+ weeks of use when you've identified a genuine need (5+ instances). Learn the 9-step process, delegation strategies, and when to use agents vs commands. Includes statistical analyst and data quality checker examples.

**[Cross-Reference Documentation](how-to/cross-reference-docs.md)**
Reference files using plain paths, NOT @ syntax. Learn 5 referencing methods (plain paths, structured references, inline+detail, directory refs, conditional), when to use each, and how to avoid context bloat. Emphasizes progressive disclosure without auto-loading.

### Reference Documentation (Information-Oriented)

**[CLAUDE.md Specification](reference/claude-md-specification.md)**
Complete CLAUDE.md syntax reference. File locations, load hierarchy, recommended sections, import system (using plain paths), size recommendations by project type, hierarchical patterns, markers/symbols, memory system, validation checklist.

**[Command Specification](reference/command-specification.md)**
Slash command technical reference. YAML frontmatter schema (all fields), argument system ($ARGUMENTS, $1-$N), allowed tools table, organization strategies (flat vs namespace), four major command patterns, security considerations, testing checklist.

**[Agent Specification](reference/agent-specification.md)**
Subagent definition reference. YAML field specifications, tool permissions, model selection (haiku/sonnet/opus), delegation system (explicit vs automatic), context isolation, four agent design patterns, multi-agent coordination, testing checklist.

**[Project Structures](reference/project-structures.md)**
Recommended directory structures by project type. Selection matrix, five base patterns (modular monolith, multi-language, data science, API, dual-track), language-specific structures, tooling-specific layouts (Nx, Bazel), migration paths, validation checklist.

**[File Path Syntax](reference/file-path-syntax.md)**
File referencing guide emphasizing NO @ syntax. Relative vs absolute paths, platform considerations, usage contexts (CLAUDE.md, commands, agents, conversation), cross-platform compatibility, common pitfalls table, quick reference.

### Explanation (Understanding-Oriented)

**[Documentation Philosophy](explanation/documentation-philosophy.md)**
AI-first vs human-first documentation. When to use direct file paths vs semantic references, why CLAUDE.md should be concise (token economics), Diátaxis framework for AI context, progressive disclosure through modularity. Debunks: "More documentation is always better."

**[Configuration Layers](explanation/configuration-layers.md)**
Why three layers and when to use each. CLAUDE.md for persistent memory (always-on context), commands for reusable workflows (on-demand procedures), subagents for specialized expertise (context isolation). Decision framework for choosing the right layer.

**[Iterative Enhancement](explanation/iterative-enhancement.md)**
Why NOT to build complex setups immediately. The One-Week Rule (use zero config first), stage-by-stage evolution timeline, benefits of gradual construction, triggers for adding complexity, The Build Trap vs The Iterate Trap. Debunks: "I should set up complete infrastructure before starting."

**[Cross-Language Projects](explanation/cross-language-projects.md)**
Multi-language repo architecture and AI context management. Why projects become polyglot (right tool for job, team expertise), data interchange patterns (Parquet, Protocol Buffers), hierarchical CLAUDE.md for language-specific contexts, build orchestration (Make → Nx → Bazel), integration testing, decision framework.

**[Modular Monolith Pattern](explanation/modular-monolith-pattern.md)**
Why modular monoliths work for AI-augmented projects. Single deployment with clear boundaries, AI benefits (hierarchical understanding, atomic changes, single context), Strangler Fig pattern (evolution to microservices when justified), module communication, extraction checklist. Debunks: "Microservices are always better at scale."

---

## Document Map by Topic

### Configuration & Setup
- Tutorial: [Getting Started](tutorials/01-getting-started.md)
- How-To: [Setup CLAUDE.md](how-to/setup-claude-md.md)
- Reference: [CLAUDE.md Specification](reference/claude-md-specification.md)
- Explanation: [Configuration Layers](explanation/configuration-layers.md)

### Commands
- Tutorial: [Progressive Enhancement](tutorials/02-progressive-enhancement.md) (Section: Commands)
- How-To: [Create a Slash Command](how-to/create-slash-command.md)
- Reference: [Command Specification](reference/command-specification.md)

### Agents
- Tutorial: [Progressive Enhancement](tutorials/02-progressive-enhancement.md) (Section: Agents)
- How-To: [Create a Specialized Agent](how-to/create-specialized-agent.md)
- Reference: [Agent Specification](reference/agent-specification.md)

### Documentation & References
- How-To: [Cross-Reference Documentation](how-to/cross-reference-docs.md)
- Reference: [File Path Syntax](reference/file-path-syntax.md)
- Explanation: [Documentation Philosophy](explanation/documentation-philosophy.md)

### Project Architecture
- Reference: [Project Structures](reference/project-structures.md)
- Explanation: [Modular Monolith Pattern](explanation/modular-monolith-pattern.md)
- Explanation: [Cross-Language Projects](explanation/cross-language-projects.md)

### Workflow & Best Practices
- Tutorial: [Progressive Enhancement](tutorials/02-progressive-enhancement.md)
- Explanation: [Iterative Enhancement](explanation/iterative-enhancement.md)

---

## Quick Reference Card

### The Three-Layer System
```
CLAUDE.md (Always-On Context)
├─ Project overview
├─ Tech stack & commands
├─ Critical conventions
└─ Import paths to details

Commands (On-Demand Workflows)
├─ Repetitive tasks
├─ Multi-step procedures
└─ Cross-language automation

Agents (Specialized Expertise)
├─ Domain-specific tasks
├─ Isolated context
└─ Automatic delegation
```

### File Referencing Rules
```
✅ CORRECT: docs/manuals/claude/reference/file.md
❌ WRONG:   @docs/manuals/claude/reference/file.md

@ syntax = Auto-loads file into context
Plain path = Informational reference only
```

### When to Add Complexity
```
Week 1-2:  Use Claude Code with ZERO config
Week 3:    Add basic CLAUDE.md (if repeating yourself)
Week 4-6:  Create 1-2 slash commands (if task is frequent)
Week 8+:   Consider agents (if need isolated expertise)
```

### Decision Framework: Command vs Agent
```
USE COMMAND when:
- Task-specific (one clear goal)
- Same workflow every time
- No specialized expertise needed
- Quick execution (<5 steps)

USE AGENT when:
- Domain expertise required
- Complex decision-making
- Need isolated context
- Multi-stage workflows
- After identifying 5+ similar needs
```

---

## Manual Statistics

**Total Documents:** 16
- **Tutorials:** 2 (learning-oriented)
- **How-To Guides:** 4 (task-oriented)
- **Reference Docs:** 5 (information lookup)
- **Explanations:** 5 (conceptual understanding)

**Estimated Reading Time:**
- Quick Start (tutorials only): 2-3 hours
- Complete Manual: 8-10 hours
- Reference lookup: <5 minutes per query

**Coverage Areas:**
- ✅ Configuration (CLAUDE.md, commands, agents)
- ✅ Project structures (monolith, multi-language, data science)
- ✅ Documentation strategies (AI-first vs human-first)
- ✅ Workflow patterns (iterative enhancement)
- ✅ Best practices (file references, naming, organization)

---

## Key Principles

### 1. **Iterative Construction (Not Immediate Setup)**
Build your Claude Code configuration gradually based on ACTUAL use, not hypothetical needs. Start with zero config, add CLAUDE.md when you notice repetition, create commands for frequent tasks, and only create agents after weeks of identifying genuine needs.

**See:** [Iterative Enhancement](explanation/iterative-enhancement.md)

### 2. **Plain File Paths (Not @ Syntax)**
Reference documentation using plain file paths like `docs/manuals/claude/file.md`. The @ syntax auto-loads files into context, which we want to avoid for informational references. Use plain paths for progressive disclosure without context bloat.

**See:** [Cross-Reference Documentation](how-to/cross-reference-docs.md), [File Path Syntax](reference/file-path-syntax.md)

### 3. **User-Driven Enhancement**
Claude Code should ASK before creating agents, commands, or complex configurations. The user drives the process iteratively based on their workflow, not automated setup scripts. Collaboration beats automation.

**See:** [Progressive Enhancement](tutorials/02-progressive-enhancement.md)

### 4. **Context Economy**
Keep CLAUDE.md concise (<100 lines for most projects). Use imports for detailed content. Focus on non-obvious, project-specific information. Let commands and agents handle specialized knowledge. Token efficiency through progressive disclosure.

**See:** [Documentation Philosophy](explanation/documentation-philosophy.md), [CLAUDE.md Specification](reference/claude-md-specification.md)

### 5. **Modular Architecture**
Structure projects with clear boundaries even in monoliths. Hierarchical CLAUDE.md files for language-specific contexts. Explicit interfaces between components. Clean extraction paths when complexity demands it.

**See:** [Modular Monolith Pattern](explanation/modular-monolith-pattern.md), [Project Structures](reference/project-structures.md)

---

## Source Attribution

This manual consolidates and synthesizes content from multiple research reports:

### Primary Sources
- **claude_best_practices.md** - Claude Code implementation guide for data science repositories, covering CLAUDE.md configuration, slash commands, specialized agents, MCP integration, and multi-language project patterns.

- **project_structure.md** - Comprehensive architectural guide for AI-augmented, cross-disciplinary technical projects, including modular monolith patterns, plugin architectures, dual-track development, and AI-first project organization.

- **documentation_excellence.md** - Professional documentation framework covering Diátaxis structure, docs-as-code principles, decision frameworks, multi-language strategies, quality scorecards, and AI documentation agent design.

### Integrated Philosophy
- **documentation-philosophy.md** - Core insights on AI-first vs human-first documentation, token efficiency, coherence vs efficiency optimization, and the hybrid strategy balancing comprehensive guides with targeted manuals.

### Consolidation Approach
Content was reorganized following the **Diátaxis framework** (tutorials, how-to guides, reference, explanation) to create cohesive, navigable documentation optimized for both Claude Code's understanding and human reference. Cross-cutting concerns were extracted and restructured around user needs rather than source document boundaries.

---

## Version Information

**Manual Version:** 1.0.0
**Last Updated:** 2025-10-31
**Claude Code Version:** Latest (2025)
**Status:** ✅ Production Ready

---

## Contributing & Feedback

This manual is a living document that evolves with Claude Code best practices.

**Found an error?** Check docs/source-reports/claude/ and suggest corrections.

**Have a better pattern?** Document it in your project, test for 2+ weeks, then propose addition.

**Documentation philosophy questions?** See [Documentation Philosophy](explanation/documentation-philosophy.md) for the underlying principles.

---

## Getting Started Path

**New to Claude Code?** Follow this 4-week learning path:

### Week 1: Foundation
1. Read [Getting Started](tutorials/01-getting-started.md) (1 hour)
2. Create basic CLAUDE.md in your project (30 min)
3. Use Claude Code with just CLAUDE.md (rest of week)

### Week 2: First Enhancements
1. Read [Progressive Enhancement](tutorials/02-progressive-enhancement.md) (1-2 hours)
2. Create 1 slash command for a repetitive task (1 hour)
3. Test and refine based on actual use (rest of week)

### Week 3-4: Deep Understanding
1. Read [Configuration Layers](explanation/configuration-layers.md) (30 min)
2. Read [Iterative Enhancement](explanation/iterative-enhancement.md) (30 min)
3. Read [Documentation Philosophy](explanation/documentation-philosophy.md) (45 min)
4. Refine your setup based on insights (ongoing)

### Week 5+: Advanced Patterns
1. Consider agents (only if you've identified 5+ similar complex needs)
2. Review [Project Structures](reference/project-structures.md) for architecture patterns
3. Explore cross-language or specialized documentation strategies as needed

**Total time investment:** ~5-7 hours over 4 weeks
**Expected productivity gain:** 5-10x on documented workflows

---

**Ready to begin?** Start with [Getting Started](tutorials/01-getting-started.md) →
