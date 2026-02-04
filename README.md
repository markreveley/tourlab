# tourlab

A tour advancing agent built on [agen](https://github.com/markreveley/agen).

Files are context, git is the audit trail, the terminal is the interface.

## What This Is

A **template** for managing concert tour logistics. Tourlab provides the agent, the rules, and the protocol. Your tour data lives separately, in its own directory and git repo.

The `agent` script wraps [agen](https://github.com/markreveley/agen) with tour-advancing context:
- SYSTEM.md defines the agent's identity and rules
- `STATE_DIR` points to your tour data (dates, venues, emails)
- The agent knows what files exist and can reference them

## Quick Start

```bash
# 1. Clone tourlab
git clone https://github.com/youruser/tourlab.git
cd tourlab

# 2. Install agen (if not already)
# See: https://github.com/markreveley/agen

# 3. Create your tour state (outside tourlab)
mkdir -p ../my-tour/state/{dates,venues,emails}
cd ../my-tour && git init

# 4. Create your first date file
cat > state/dates/2026-03-15-boston.md << 'EOF'
---
city: Boston
venue: House of Blues
status: unconfirmed
load_in_time: null
sound_check_time: null
doors_time: null
set_time: null
curfew: null
---

## Notes

## Outstanding
- [ ] Confirm date

## Evidence Log
EOF

# 5. Use
STATE_DIR=../my-tour/state ../tourlab/agent "what dates need attention?"
cat email.txt | STATE_DIR=../my-tour/state ../tourlab/agent "extract info for the boston date"
```

## State lives outside tourlab

Tourlab is a template you can update (`git pull`) without touching your data. Your tour state is a separate directory with its own git history.

```
AGEN/
├── tourlab/              # Template (this repo)
│   ├── agent             # Wrapper script (calls agen)
│   └── SYSTEM.md         # Agent identity + evidence protocol
└── my-tour/              # Your state (separate git repo)
    └── state/
        ├── dates/        # Per-date files (YYYY-MM-DD-city.md)
        ├── venues/       # Reusable venue info
        ├── emails/       # Source documents (agent reads, never modifies)
        ├── context.md    # Working memory
        └── TODO.md       # Outstanding questions
```

Set `STATE_DIR` to tell the agent where your state lives:

```bash
# Explicit
STATE_DIR=/path/to/my-tour/state ./agent "query"

# Or export it
export STATE_DIR=/path/to/my-tour/state
./agent "query"
```

## Requirements

- [agen](https://github.com/markreveley/agen) in PATH
- One of agen's backends (Claude Code, llm CLI, or API key)

## Usage

```bash
# Ask questions
STATE_DIR=../my-tour/state ./agent "what dates are missing tech specs?"

# Process an email
cat incoming-email.txt | STATE_DIR=../my-tour/state ./agent "extract relevant info for boston"

# Summarize
STATE_DIR=../my-tour/state ./agent "what needs attention this week?"

# Pipeline
STATE_DIR=../my-tour/state ./agent "list all null fields" | grep "load_in"
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
git -C ../my-tour log --oneline -10

# What fields are still null for Boston?
grep ": null" ../my-tour/state/dates/2026-03-15-boston.md

# What evidence do we have?
grep -A3 "^### " ../my-tour/state/dates/*.md

# Find dates missing load-in times
grep -l "load_in_time: null" ../my-tour/state/dates/*.md
```

## Adding a New Date

1. Create a new file in `state/dates/` named `YYYY-MM-DD-city.md`
2. Add YAML frontmatter with all fields set to `null`
3. Set status to `unconfirmed`, `confirmed`, or `advancing`
4. Add outstanding questions to the Outstanding section

## Adding Emails

1. Copy email content to `state/emails/descriptive-name.md`
2. Include headers and timestamps
3. Run: `cat state/emails/name.md | STATE_DIR=../my-tour/state ./agent "extract info for [city]"`

## How It Works

The `agent` script is pure shell. It builds context deterministically (no LLM), then calls agen:

```bash
# What agent does (simplified):
{
  cat SYSTEM.md
  cat $STATE_DIR/context.md
  ls $STATE_DIR/dates/*.md $STATE_DIR/venues/*.md
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
