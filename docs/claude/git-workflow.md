# Git Commit Philosophy

**Descriptive commits with clear title and detailed body. No fluff.**

## Commit Message Format

```
Title line (capitalized, imperative mood, concise)

Organized explanation of changes in 2-4 logical sections:
- Thorough but concise - clarity over comprehensiveness
- Use bullets to highlight key points and improvements
- Explain WHAT changed and WHY, not HOW (code shows how)
- NO "authored by", "generated with", or bot signatures
```

## Good Example

```
Add comprehensive documentation system

Created hierarchical documentation with AI context and technical reference:

Documentation Structure:
- Claude.md files: Project overview, source organization, subsystem guides
- /docs/ directory: Architecture, features, config, UI, contributing guides

Key Features:
- Layered detail: High-level overview in claude.md, technical depth in docs/
- Cross-referenced with update reminders for code changes
- Workflow-focused for both AI assistants and human developers
```

## Bad Examples

```
✗ WIP: working on calculator stuff

✗ feat: add calculator enhancements (#123)

✗ Update files
  🤖 Generated with Claude Code
  Co-Authored-By: Bot <bot@example.com>
```

## Guidelines

- **Title**: Capitalized, imperative mood ("Add feature" not "Added feature")
- **Body**: 2-4 organized sections with bullets - thorough but concise
- **Length**: Detailed enough to understand changes, not exhaustive documentation
- **No fluff**: No bot signatures, no "generated with" messages
- **Focused**: One logical change per commit when possible
- **Purpose-driven**: Explain what changed and why, not implementation details

## Using the `/commit` Command

The project includes a `/commit` slash command that follows these guidelines automatically:

```bash
/commit                    # Analyze all changes and create commit
/commit focus on WezTerm   # Create commit with specific focus
/commit amend              # Amend previous commit (with safety checks)
```

The command will:
1. Analyze your changes
2. Draft a properly formatted commit message
3. Show preview for approval
4. Execute git commands

## Amending Commits

**Safety First:**
- Only amend commits you authored
- Never amend commits already pushed to shared branches
- The `/commit amend` command checks these automatically

## Branch Strategy

- **feature/** branches for new features
- **fix/** branches for bug fixes
- Merge to main when complete and tested
