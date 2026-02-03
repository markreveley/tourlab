# API vs CLI: Inference Economics for Vision B

This document explains why routing through Claude Code with `--tools ""` is an unexpectedly good fit for Unix-shaped agents.

## The Standard Value Propositions

**Anthropic API:**
- Pay per token (input + output)
- Full control over requests
- Build your own tooling
- API key required (credential to manage)

**Claude Code (for 99% of users):**
- Flat monthly fee (Max subscription)
- LLM + Tools = agentic coding assistant
- Interactive TUI experience
- OAuth-based auth (no API key)

## The Vision B Inversion

Vision B (agent-in-the-shell) explicitly rejects tool use. The agent should be a Unix filter:
- stdin → LLM → stdout
- No autonomous actions
- Outer script controls context
- Composable in pipelines

This means Claude Code's primary value proposition (tools) is exactly what we *don't* want.

## The Arbitrage

By running Claude Code with `--tools ""`, we get:

```
Claude Code (Max subscription)
       │
       ▼
  ┌─────────────────┐
  │  --tools ""     │  ← Disable the "premium" feature
  └────────┬────────┘
           │
           ▼
  Pure LLM inference
  (exactly what Vision B needs)
```

**What we're paying for:** Unlimited inference + tools
**What we're using:** Unlimited inference only

## Cost Comparison

| Approach | Cost Model | API Key Required | Tool Use |
|----------|-----------|------------------|----------|
| Direct API | Per-token | Yes (leak risk) | Build it yourself |
| `llm` CLI | Per-token (via API) | Yes | No |
| Claude Code + `--tools ""` | Flat monthly (Max) | No (OAuth) | Disabled |

For moderate-volume interactive use, Max subscription is likely cheaper than API — and you get the security benefit of no API key to leak.

## The Moltbook Lesson

In early 2026, the Moltbook platform exposed 1.5M API keys through a misconfigured database. Users had stored their OpenAI/Anthropic keys in the platform, creating a single point of failure.

Key insight: Any system where API keys are stored as strings (in configs, databases, environment variables) is vulnerable to:
- Accidental commits to public repos
- Database breaches
- Log exposure
- Environment variable leaks

Claude Code sidesteps this entirely. Authentication is OAuth-based — there's no API key in your possession to leak.

## Alignment with Vision B

| Vision B Requirement | Claude Code + `--tools ""` |
|---------------------|---------------------------|
| Pure text in/out | ✓ Tools disabled |
| No agent autonomy | ✓ Can't take actions |
| Composable in pipelines | ✓ stdin/stdout work |
| Affordable at volume | ✓ Flat fee vs per-token |
| No credentials to leak | ✓ OAuth-based auth |
| Predictable behavior | ✓ No tool hallucination |

## Trade-offs and Limitations

### Rate limits
Max subscription has rate limits (generous but not unlimited). For high-volume batch processing, API might be necessary.

### Runtime overhead
Claude Code is heavier than `llm` CLI:
- Startup time is longer
- More dependencies
- Larger process footprint

For quick one-off queries, `llm` CLI might feel snappier.

### Dependency on Anthropic infrastructure
Using Claude Code means depending on Anthropic's OAuth and subscription systems. Direct API is more self-contained.

### Model selection
Claude Code uses whatever model is current. API lets you pin specific model versions for reproducibility.

## When to Use What

| Scenario | Recommended |
|----------|-------------|
| Interactive development | Claude Code + `--tools ""` |
| Moderate pipeline use | Claude Code + `--tools ""` |
| High-volume batch processing | API (for rate limits) |
| Reproducibility-critical | API (pin model version) |
| Quick one-liners | `llm` CLI (if already configured) |
| No API key management | Claude Code |

## Configuration

In `agent.conf`:

```bash
# Use Claude Code (recommended for most use cases)
AGENT_BACKEND=claude-code

# Fallback to llm CLI if needed
# AGENT_BACKEND=llm

# Fallback to direct API if needed
# AGENT_BACKEND=api
```

The agent script handles the routing:

```bash
# Claude Code path (in agent script)
echo "$prompt" | claude -p --tools "" --output-format text
```

## Summary

Claude Code with `--tools ""` is a fortuitous fit for Vision B:

1. **Economics:** Flat fee beats per-token for moderate use
2. **Security:** No API key to leak
3. **Philosophy:** Disabling tools aligns with Unix filter paradigm
4. **Predictability:** No risk of autonomous actions

We're paying for a power tool and using it as a hammer — but for Vision B, that's exactly the right tool.
