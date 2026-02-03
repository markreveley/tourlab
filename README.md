# tourlab

A Unix-shaped tour advancing agent. Files are context, git is the audit trail, the terminal is the interface.

## Philosophy

This is a thesis validation project for "Unix Agentics" — the idea that agents should be files, processes, and streams, not chat interfaces.

**Core beliefs:**

- **Files are the context substrate.** Not databases, not vector stores — markdown files you can grep.
- **Git is the audit trail.** Every state change is a commit. `git log` tells you what happened and when.
- **The agent is a CLI tool.** Scriptable, composable, observable with standard Unix tools.
- **Evidence over inference.** No field changes without source documents and quotes.
- **Observability by construction.** `find`, `grep`, `cat`, `git log` — no custom dashboards needed.

## Domain: Tour Advancing

Tour advancing coordinates logistics for concert dates. Each date progresses through:

```
unconfirmed → confirmed → advancing → complete
```

The agent helps extract information from emails and maintain accurate state files with full evidence trails.

## Structure

```
tourlab/
├── .claude/
│   └── skills/         # Custom Claude Code slash commands
│       └── ship.md     # /ship - commit with README enforcement
├── state/
│   ├── dates/           # Per-date files (YYYY-MM-DD-city.md)
│   ├── venues/          # Reusable venue info (venue-slug.md)
│   ├── emails/          # Source documents (read-only to agent)
│   ├── context.md       # Agent's working memory
│   └── TODO.md          # Outstanding items and questions
├── agent               # CLI entry point
├── agent.conf          # Backend configuration
├── SYSTEM.md           # System prompt + edit protocol
├── CLAUDE.md           # Instructions for Claude Code
└── README.md
```

## Usage

```bash
# Ask questions about current state
./agent "what dates are missing tech specs?"

# Process an email
cat state/emails/boston-tech.md | ./agent "extract relevant info for boston"

# Summarize work needed
./agent "what needs attention this week?"

# Pipe workflows
./agent "list all null fields for confirmed dates" | grep "load_in"
```

## Requirements

One of these backends:
- **Claude Code** — `claude` CLI (uses Max subscription, no API costs)
- **llm CLI** — from [llm.datasette.io](https://llm.datasette.io) — `pip install llm` (API costs)
- **Direct API** — `ANTHROPIC_API_KEY` environment variable (API costs)

And: `jq` (for direct API calls)

## Configuration

Set the backend in `agent.conf`:

```bash
AGENT_BACKEND=claude-code   # Uses Max subscription via Claude Code (default)
AGENT_BACKEND=llm           # Uses llm CLI (API costs)
AGENT_BACKEND=api           # Uses direct Anthropic API (API costs)
AGENT_BACKEND=auto          # Auto-detect available backend
```

Or override with environment variable:

```bash
AGENT_BACKEND=llm ./agent "what needs attention?"
```

## Debugging

Inspect the prompt without sending to the LLM:

```bash
DEBUG=1 ./agent "your task"              # print full prompt
DEBUG=1 ./agent "your task" > prompt.md  # save for inspection
```

## The Evidence Protocol

Every field change must have an evidence entry:

```markdown
### 2026-01-28 — load_in_time set to 14:00
Source: emails/boston-production-thread.md
Quote: "Load in will be at 2pm, please have trucks at loading dock by 1:45"
Changed: `load_in_time: null` → `load_in_time: 14:00`
```

No guessing. No inference. If it's ambiguous, it goes to `TODO.md` as a question.

## Observability

```bash
# What changed recently?
git log --oneline -10

# What fields are still null for Boston?
grep ": null" state/dates/2026-03-15-boston.md

# What evidence do we have?
grep -A3 "^### " state/dates/*.md

# What's outstanding?
cat state/TODO.md

# Find all dates missing load-in times
grep -l "load_in_time: null" state/dates/*.md
```

## Adding a New Date

1. Copy an existing date file as template
2. Update frontmatter fields
3. Set status to `unconfirmed` or `confirmed`
4. Leave unknown fields as `null`
5. Add outstanding questions to the Outstanding section

## Adding Emails

1. Copy email thread content to `state/emails/descriptive-name.md`
2. Include full headers and timestamps
3. Mark as read-only (agent reads but never modifies)
4. Run agent with the email as input to extract information

## Development

### Committing changes

Use `/ship` in Claude Code to commit with documentation enforcement:

1. `/ship` triggers semantic review — Claude verifies README matches code changes
2. Git pre-commit hook provides syntactic backstop — blocks if README wasn't touched

This two-layer approach ensures documentation stays current.

### Setting up the pre-commit hook

The hook isn't version-controlled (lives in `.git/`). To set it up:

```bash
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
```

## Why This Way?

Chat interfaces are session-isolated and ephemeral. This approach:

- **Persists** — State survives sessions. Pick up where you left off.
- **Composes** — Pipe agents together. Script workflows. Integrate with other tools.
- **Audits** — Every change has a commit. Every field has evidence.
- **Observes** — Debug with grep, not dashboards.

The terminal is a 50-year-old prototype of an agentic interface. This project takes that seriously.
