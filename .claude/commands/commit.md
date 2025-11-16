# Description: Create a git commit following project commit philosophy

Create a professional git commit following the project's standards.

**Usage:**
- `/commit` - Analyze all changes and create commit
- `/commit [focus]` - Create commit with specific focus (e.g., "only WindowDimmer changes")
- `/commit add to last commit` - Amend previous commit (detects phrases like "amend", "add to last", "forgot to include")
- `/commit merge [branch]` - Merge branch with proper history preservation (detects phrases like "merge", "merge with main", "merge branch")

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
   - Check if user wants to merge (phrases: "merge", "merge with", "merge branch")
   - Check if user wants to amend (phrases: "amend", "add to last", "forgot to include", "append to previous")
   - If merging: Jump to Merge Process below
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

---

## Merge Process (when detected)

**IMPORTANT:** Always use `--no-ff` (no fast-forward) to preserve branch history and show that work was done on a branch.

1. **Determine branch to merge:**
   - Check current branch with `git branch`
   - Identify source branch (branch to merge from)
   - Identify target branch (branch to merge into, usually main)

2. **Verify branches are ready:**
   - Run `git log <target>..<source> --oneline` to see commits being merged
   - Run `git log <source>..<target> --oneline` to check if target has new commits
   - Run `git status` to ensure working tree is clean

3. **Generate merge commit message:**
   - Title format: `Merge branch '<branch-name>': <brief description>`
   - Body sections summarizing the branch work:
     - Main feature/changes added
     - Technical details and implementation notes
     - Documentation updates
     - Any additional changes (like gitignore updates)

4. **Execute merge:**
   - `git checkout <target-branch>` (if not already there)
   - `git merge --no-ff <source-branch> -m "<message>"` using HEREDOC format:
     ```bash
     git merge --no-ff branch-name -m "$(cat <<'EOF'
     Merge branch 'branch-name': Brief description

     Section 1:
     - Key point 1
     - Key point 2

     Section 2:
     - Implementation detail 1
     - Implementation detail 2
     EOF
     )"
     ```
   - Show `git log --oneline --graph -5` to confirm branch history is visible

5. **Verify merge:**
   - Confirm graph shows branch structure (should see merge lines)
   - Check that branch commits are preserved in history

## Good Merge Example

```
git merge --no-ff vd-hotkey -m "$(cat <<'EOF'
Merge branch 'vd-hotkey': Virtual desktop window management

Integrated virtual desktop feature with keyboard shortcuts:

Virtual Desktop Feature:
- Win+Alt+Left/Right hotkeys for moving windows between desktops
- Circular navigation with automatic wrapping
- System window protection and validation

VirtualDesktop Library:
- Enhanced DesktopActions.ahk2 with relative desktop navigation
- COM interface integration for Win10/Win11 compatibility

Documentation:
- Complete virtual-desktop-api.md reference
- Updated features.md with WindowVirtualDesktop details
EOF
)"
```

After merge, graph should show:
```
*   1a36729 Merge branch 'vd-hotkey': Virtual desktop window management
|\
| * 505d596 Add Lite editor project file to gitignore
| * 9e28dc2 Add virtual desktop window management feature
|/
```

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
