# Claude Code: Skills, Hooks, and Enforcement Patterns

This document explains the mechanisms available for customizing Claude Code behavior and their enforcement characteristics.

## The Enforcement Spectrum

When working with Claude Code, you have several mechanisms to influence behavior. They differ in **where they run** and **how hard they enforce**.

```
Soft ◄──────────────────────────────────────────────► Hard

CLAUDE.md    Skills    Claude Hooks    Git Hooks    Exit codes
   │            │            │              │            │
   ▼            ▼            ▼              ▼            ▼
Instructions  Injected    Shell cmds    Shell cmds   Process
to LLM        prompts     on events     on commit    terminates
```

## Mechanism Comparison

| Mechanism | What it is | When it runs | Enforcement | Can block? |
|-----------|-----------|--------------|-------------|------------|
| CLAUDE.md | Markdown instructions | Loaded at session start | Soft | No |
| Skills (slash commands) | Markdown prompts | When invoked by user | Soft | No |
| Claude Code hooks | Shell commands | On tool events | Medium | Warns only |
| Git pre-commit hooks | Shell scripts | Before `git commit` | Hard | **Yes** |
| Git pre-push hooks | Shell scripts | Before `git push` | Hard | **Yes** |

## Soft Enforcement: LLM Instructions

### CLAUDE.md

A markdown file loaded into Claude's context at session start. Lives at:
- `~/.claude/CLAUDE.md` (global)
- `./CLAUDE.md` (project-level)

**Characteristics:**
- Read once at session start
- Influences LLM behavior through instructions
- Can be forgotten or deprioritized mid-session
- No enforcement mechanism — purely advisory

**Use for:** Coding style, project conventions, general guidance

### Skills (Slash Commands)

Markdown files that inject prompts when invoked. Lives at:
- `~/.claude/skills/*.md` (global — must be here to work)
- `.claude/skills/*.md` (project-level, for version control, must be copied to global)

**Characteristics:**
- Explicitly invoked by user (`/skillname`)
- Injected into context at invocation time
- Still just instructions to the LLM
- No automatic enforcement

**Use for:** Workflows, checklists, multi-step procedures

**Important:** Skills must be in `~/.claude/skills/` to be discovered. Project-level skills in `.claude/skills/` are for version control — copy them to the global location:

```bash
cp .claude/skills/*.md ~/.claude/skills/
```

Restart the session after adding new skills.

## Medium Enforcement: Claude Code Hooks

Shell commands that run in response to Claude Code events. Configured in:
- `~/.claude/hooks.json`

**Characteristics:**
- Run automatically on events (postToolUse, preToolUse, etc.)
- Can print warnings that Claude sees
- Cannot block tool execution (tool already ran for postToolUse)
- preToolUse hooks can return errors, but behavior varies

**Configuration example:**

```json
{
  "hooks": [
    {
      "event": "postToolUse",
      "tools": ["Edit", "Write"],
      "command": "echo '⚠️ Remember to update README if needed'"
    }
  ]
}
```

**Events available:**
- `preToolUse` — Before a tool runs
- `postToolUse` — After a tool completes

**Use for:** Reminders, logging, notifications

## Hard Enforcement: Git Hooks

Shell scripts that git runs at specific points. Live in `.git/hooks/` (not version-controlled).

**Characteristics:**
- Run by git, not Claude Code
- Exit code determines success/failure
- `exit 1` blocks the operation
- Cannot be bypassed by LLM behavior (only by `--no-verify`)

**Key hooks:**
- `pre-commit` — Runs before commit is created
- `pre-push` — Runs before push to remote
- `commit-msg` — Can validate/modify commit message

**Use for:** Invariants that must hold, quality gates, format checks

## The Semantic vs Syntactic Gap

A critical distinction when designing enforcement:

| Check Type | Question | Who can answer |
|------------|----------|----------------|
| Syntactic | "Was file X modified?" | Shell script |
| Semantic | "Does the modification make sense?" | Human or LLM |

Git hooks can only do syntactic checks:
- File exists/doesn't exist
- File was modified/wasn't modified
- Pattern matches/doesn't match

Git hooks **cannot** verify:
- Code change is correctly documented
- README update is relevant to code change
- Documentation is accurate

This is why we use two layers:

1. **Skill (`/ship`)** — Does semantic verification (LLM reviews relationship between code and docs)
2. **Git hook** — Does syntactic backstop (was README touched at all?)

## This Project's Pattern

### `/ship` skill (semantic layer)

Located: `~/.claude/skills/ship.md` (global) and `.claude/skills/ship.md` (version controlled)

When invoked:
1. Checks `git status` and `git diff`
2. Categorizes changes (code vs docs)
3. Reviews whether README accurately documents code changes
4. Updates README if needed
5. Commits with descriptive message

**Enforcement:** Soft — relies on LLM following instructions

### Pre-commit hook (syntactic layer)

Located: `.git/hooks/pre-commit` (local only)

When `git commit` runs:
1. Lists staged files
2. Checks if any "code" files are staged
3. Checks if README.md is staged
4. Blocks commit if code changed without README

**Enforcement:** Hard — git refuses to commit

### The combination

```
User types /ship
       │
       ▼
┌─────────────────┐
│  /ship skill    │  Semantic: "Does README match code?"
│  (soft)         │  If no → update README
└────────┬────────┘
         │
         ▼
    git commit
         │
         ▼
┌─────────────────┐
│ pre-commit hook │  Syntactic: "Was README touched?"
│ (hard)          │  If no → exit 1, block commit
└────────┬────────┘
         │
         ▼
   Commit created
```

The skill prevents bad documentation. The hook prevents missing documentation.

## Setup Checklist

### For new clones of this project:

```bash
# 1. Copy skills to global location
mkdir -p ~/.claude/skills
cp .claude/skills/*.md ~/.claude/skills/

# 2. Set up git hook
cat > .git/hooks/pre-commit << 'EOF'
#!/usr/bin/env bash
set -euo pipefail
staged=$(git diff --cached --name-only)
code=$(echo "$staged" | grep -vE '\.(md|json)$|^\.gitignore$|^\.' | head -1 || true)
readme=$(echo "$staged" | grep -q '^README\.md$' && echo yes || true)
if [[ -n "$code" && -z "$readme" ]]; then
    echo "ERROR: Code changed but README.md not updated"
    exit 1
fi
EOF
chmod +x .git/hooks/pre-commit

# 3. Restart Claude Code session
```

### Verifying setup:

```bash
# Check skill is available
ls ~/.claude/skills/ship.md

# Check hook is executable
ls -la .git/hooks/pre-commit

# Test hook (should fail)
echo "test" >> agent.conf
git add agent.conf
git commit -m "test"  # Should be blocked
git restore --staged agent.conf
git checkout agent.conf
```

## When to Use Each Mechanism

| Goal | Mechanism |
|------|-----------|
| "Follow this coding style" | CLAUDE.md |
| "When committing, check X" | Skill |
| "Remind me after edits" | Claude Code hook |
| "Never commit without X" | Git hook |
| "Never push failing tests" | Git pre-push hook |

## Limitations

### Skills can be ignored
The LLM might not follow skill instructions perfectly. They're guidance, not guarantees.

### Claude Code hooks can't block
They run after the fact (postToolUse) or their blocking behavior is inconsistent (preToolUse).

### Git hooks can be bypassed
`--no-verify` skips hooks. This is sometimes necessary but should be rare.

### Semantic checks require LLM or human
No shell script can verify "documentation is correct." This will always be a soft layer.

## Future Considerations

- **CI/CD enforcement:** Move syntactic checks to CI for team-wide enforcement
- **LLM-powered hooks:** Use cheap/fast models for semantic pre-commit checks (adds latency/cost)
- **Hook generators:** Script that sets up all hooks from a config file in the repo
