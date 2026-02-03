# /ship - Commit checkpoint with README enforcement

When the user invokes /ship, follow these steps strictly:

## Step 1: Check what changed

Run `git status` and `git diff --name-only` to see all modified files.

## Step 2: Categorize changes

Identify:
- **Code files**: Scripts, config files (agent, agent.conf, *.sh, etc.)
- **Documentation**: README.md, CLAUDE.md, other .md files

## Step 3: Semantic README check

If ANY code files were modified:

1. Read the current README.md
2. Review what code changes were made
3. Ask yourself: "Does README accurately document these changes?"
4. If NO: Update README first, then proceed
5. If YES: Proceed to commit

This is a HARD REQUIREMENT. Do not skip this check.

## Step 4: Commit

Stage relevant files and commit with a descriptive message.

## Step 5: Report

Show:
- What was committed
- Current git status
- How many commits ahead of origin

## Important

- Do NOT just add a blank line to README to pass the hook
- The README update must be SEMANTICALLY relevant to the code changes
- If the code change truly doesn't need documentation (e.g., typo fix), explain why
