# tourlab

A tour advancing agent built on [agen](https://github.com/markreveley/agen).

Files are context, git is the audit trail, the terminal is the interface.

## What This Is

A **template** for managing concert tour logistics. Tourlab provides the agent, the rules, and the protocol. Your tour data lives separately, in its own directory and git repo.

The `agent` script wraps [agen](https://github.com/markreveley/agen) with tour-advancing context:
- SYSTEM.md defines the agent's identity and rules
- `STATE_DIR` points to your tour data (shows, venues, personnel)
- The agent knows what files exist and can reference them

## Quick Start

```bash
# 1. Clone tourlab
git clone https://github.com/youruser/tourlab.git
cd tourlab

# 2. Install agen (if not already)
# See: https://github.com/markreveley/agen

# 3. Create your tour state (outside tourlab)
mkdir -p ../my-tour/state/{venues,personnel}
cd ../my-tour && git init

# 4. Create your first show
mkdir -p state/3-15-26-Boston/{email,assets}
cat > state/3-15-26-Boston/3-15-26-Boston.md << 'EOF'
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
        ├── 3-15-26-Boston/          # One folder per show
        │   ├── 3-15-26-Boston.md    # Show state (frontmatter + evidence log)
        │   ├── email/               # Email threads for this show
        │   └── assets/              # Riders, stage plots, contracts
        ├── 3-17-26-Philadelphia/
        │   ├── 3-17-26-Philadelphia.md
        │   ├── email/
        │   └── assets/
        ├── venues/                  # Shared venue database
        │   └── house-of-blues-boston.md
        ├── personnel/               # Contact & profile database
        │   ├── sarah-chen.md
        │   └── mike-torres.md
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
Source: email/production-thread.md
Quote: "Load in will be at 2pm, please have trucks at loading dock by 1:45"
Changed: `load_in_time: null` → `load_in_time: 14:00`
```

No guessing. No inference. If it's ambiguous, it goes to TODO.md as a question.

## Observability

```bash
# What changed recently?
git -C ../my-tour log --oneline -10

# What fields are still null for Boston?
grep ": null" ../my-tour/state/3-15-26-Boston/3-15-26-Boston.md

# What evidence do we have?
grep -A3 "^### " ../my-tour/state/*//*.md

# Find shows missing load-in times
grep -rl "load_in_time: null" ../my-tour/state/
```

## Adding a New Show

1. Create a folder in `state/` named `M-D-YY-City` (e.g., `3-15-26-Boston`)
2. Create `email/` and `assets/` subdirectories inside it
3. Create a state file named the same as the folder (e.g., `3-15-26-Boston.md`)
4. Add YAML frontmatter with all fields set to `null`
5. Set status to `unconfirmed`, `confirmed`, or `advancing`
6. Add outstanding questions to the Outstanding section

## Adding Emails

1. Copy email content to the show's `email/` folder (e.g., `state/3-15-26-Boston/email/production-thread.md`)
2. Include headers and timestamps
3. Run: `cat state/3-15-26-Boston/email/thread.md | STATE_DIR=../my-tour/state ./agent "extract info for boston"`

## Venues

Venue files live in `state/venues/` and are shared across shows. Named `venue-name-city.md`.

```yaml
---
name: House of Blues Boston
slug: house-of-blues-boston
city: Boston
state: MA
capacity: 2400
stage_width: 48ft
stage_depth: 32ft
trim_height: 26ft
house_console: Avid S6L
monitor_console: Avid S6L
house_pa: d&b audiotechnik V-Series
power_available: 400A 3-phase
production_manager: null
production_email: null
---
```

Show state files reference venues by name. Venue files persist across tours — tech specs and permanent contacts don't change per show.

## Personnel

Personnel files live in `state/personnel/`, one file per person. Named `firstname-lastname.md`.

Covers touring crew, artist/management, venue contacts, local sellers, promoter reps, runners, catering vendors — anyone the tour interacts with.

```yaml
---
name: Sarah Chen
role: Production Manager
org: House of Blues Boston
phone: null
email: null
---

## Notes
```

## How It Works

The `agent` script is pure shell. It builds context deterministically (no LLM), then calls agen:

```bash
# What agent does (simplified):
{
  cat SYSTEM.md
  cat $STATE_DIR/context.md
  ls -d $STATE_DIR/*/          # list show folders
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
