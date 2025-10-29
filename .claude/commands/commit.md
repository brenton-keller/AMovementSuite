# Description: Create a git commit following project commit philosophy

Create a professional git commit following the project's standards.

**Usage:**
- `/commit` - Analyze all changes and create commit
- `/commit [focus]` - Create commit with specific focus (e.g., "only WindowDimmer changes")
- `/commit add to last commit` - Amend previous commit (detects phrases like "amend", "add to last", "forgot to include")

## Commit Format

**Title:** Capitalized, imperative mood, concise ("Add feature" not "added feature")

**Body:**
- Blank line after title
- Organized into 2-4 logical sections with clear headers
- Bullets for key points - be thorough but concise
- Explain WHAT and WHY, not HOW
- **Length:** Detailed but not verbose - aim for clarity over comprehensiveness
- NO bot signatures, NO "generated with" fluff

## Process

1. **Detect intent:**
   - Check if user wants to amend (phrases: "amend", "add to last", "forgot to include", "append to previous")
   - If amending: Jump to Amend Process below
   - Otherwise: Continue with normal commit

2. Run `git status` and `git diff` to understand changes

3. Determine commit scope:
   - If user specified focus (e.g., "only WindowDimmer"), use that
   - Otherwise, ask user which files to commit and the main theme

4. Generate commit message:
   - Clear title summarizing the change
   - 2-4 organized sections in body (not exhaustive lists)
   - Focus on key improvements and their purpose

5. Show preview, get approval

6. Execute: `git add` files, create commit, show `git log -1 --oneline`

---

## Amend Process (when detected)

1. **Safety checks:**
   - Run `git log -1 --format='%an %ae'` to verify authorship
   - Run `git status` to check if last commit was pushed
   - **STOP if:** Last commit is not yours OR already pushed to remote

2. Show what will be amended:
   - Display last commit message
   - Show files currently in last commit
   - Show new files that will be added

3. Confirm with user:
   ```
   This will add X files to your last commit:
   "Title of last commit"

   New files: file1.txt, file2.txt

   Proceed with amend? [Yes/No]
   ```

4. Execute:
   - `git add` specified files
   - `git commit --amend --no-edit` (keeps original message)
   - Show `git log -1 --stat` to confirm

## Good Commit Example

```
Add live window resizing with performance optimizations

Implemented real-time window updates during drag operations:

Smart Update System:
- Timer-based window updates with motion pause detection
- Frame change detection prevents redundant operations
- Size thresholds block updates when window too small

Visual Feedback:
- Preview hides when window auto-updates (stationary + large enough)
- Preview stays visible when too small as warning indicator

Flash Reduction:
- Final sync only fires if position differs from target
- Eliminates redundant hide/show operations
```

## Bad Example

```
WIP: working on stuff

Fixed some bugs and added features.
🤖 Generated with Claude Code
```

## Key Rules

- Organized sections, not wall of text
- Thorough but concise - quality over quantity
- One logical change per commit
- NO fluff whatsoever
