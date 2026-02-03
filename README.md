# tourlab

A tour advancing agent built on [agen](https://github.com/markreveley/agen).

Files are context, git is the audit trail, the terminal is the interface.

## What This Is

A **template repository** for managing concert tour logistics. Clone it, replace the example data with your tour, commit your changes.

The `agent` script wraps [agen](https://github.com/markreveley/agen) with tour-advancing context:
- SYSTEM.md defines the agent's identity and rules
- state/ contains your tour data (dates, venues, emails)
- The agent knows what files exist and can reference them

## Quick Start

```bash
# 1. Clone
git clone https://github.com/youruser/tourlab.git my-tour
cd my-tour

# 2. Install agen (if not already)
# See: https://github.com/markreveley/agen

# 3. Replace example data
rm state/dates/*.md state/venues/*.md
# Add your own dates and venues

# 4. Use
./agent "what dates need attention?"
cat email.txt | ./agent "extract info for the boston date"
```

## Structure

```
tourlab/
├── agent              # Wrapper script (calls agen)
├── SYSTEM.md          # Agent identity + evidence protocol
├── state/
│   ├── dates/         # Per-date files (YYYY-MM-DD-city.md)
│   ├── venues/        # Reusable venue info
│   ├── emails/        # Source documents (agent reads, never modifies)
│   ├── context.md     # Working memory
│   └── TODO.md        # Outstanding questions
└── README.md
```

The `state/` directory ships with **example data**. Replace it with your tour.

## Requirements

- [agen](https://github.com/markreveley/agen) in PATH
- One of agen's backends (Claude Code, llm CLI, or API key)

## Usage

```bash
# Ask questions
./agent "what dates are missing tech specs?"

# Process an email
cat state/emails/boston-tech.md | ./agent "extract relevant info for boston"

# Summarize
./agent "what needs attention this week?"

# Pipeline
./agent "list all null fields" | grep "load_in"
```

## The Evidence Protocol

Every field change must have an evidence entry:

```markdown
### 2026-01-28 — load_in_time set to 14:00
Source: emails/boston-production-thread.md
Quote: "Load in will be at 2pm, please have trucks at loading dock by 1:45"
Changed: `load_in_time: null` → `load_in_time: 14:00`
```

No guessing. No inference. If it's ambiguous, it goes to TODO.md as a question.

## Observability

```bash
# What changed recently?
git log --oneline -10

# What fields are still null for Boston?
grep ": null" state/dates/2026-03-15-boston.md

# What evidence do we have?
grep -A3 "^### " state/dates/*.md

# Find dates missing load-in times
grep -l "load_in_time: null" state/dates/*.md
```

## Adding a New Date

1. Copy an existing date file as template
2. Update frontmatter fields
3. Set status to `unconfirmed`, `confirmed`, or `advancing`
4. Leave unknown fields as `null`
5. Add outstanding questions to the Outstanding section

## Adding Emails

1. Copy email content to `state/emails/descriptive-name.md`
2. Include headers and timestamps
3. Run: `cat state/emails/name.md | ./agent "extract info for [city]"`

## How It Works

The `agent` script is pure shell. It builds context deterministically (no LLM), then calls agen:

```bash
# What agent does (simplified):
{
  cat SYSTEM.md
  cat state/context.md
  ls state/dates/*.md state/venues/*.md
} > context.tmp

agen --system=context.tmp "$@"
```

The LLM only gets called at the end, via agen. Everything before that is just file concatenation.

## Why This Way?

Chat interfaces are session-isolated and ephemeral. This approach:

- **Persists** — State survives sessions
- **Composes** — Pipe agents, script workflows
- **Audits** — Every change is a commit, every field has evidence
- **Observes** — Debug with grep, not dashboards

## License

MIT
