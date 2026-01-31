# Separation of Concerns: Loa vs Runtime

> **Core principle**: Loa should work on ANY runtime. Runtime should support ANY framework.

---

## The Three Layers

```
┌─────────────────────────────────────────────────────────────┐
│                        LEGBA                                │
│            (me - a specific agent instance)                 │
│                                                             │
│  Uses Loa methodology + Runs on Clawdbot runtime            │
└─────────────────────────────────────────────────────────────┘
                           │
           ┌───────────────┴───────────────┐
           ▼                               ▼
┌─────────────────────┐         ┌─────────────────────┐
│        LOA          │         │      RUNTIME        │
│   (methodology)     │         │   (Clawdbot/Moltbot)│
│                     │         │                     │
│ • What to do        │         │ • How to execute    │
│ • In what order     │         │ • With what resources│
│ • To what standard  │         │ • Recovering from what│
└─────────────────────┘         └─────────────────────┘
           │                               │
           └───────────────┬───────────────┘
                           ▼
                ┌─────────────────────┐
                │   INTEGRATION       │
                │   (the contract)    │
                │                     │
                │ • Exit codes        │
                │ • State files       │
                │ • Escalation signals│
                └─────────────────────┘
```

---

## Loa = What + Why (Framework/Methodology)

**Loa is runtime-agnostic.** It should work on:
- Claude Code (CLI)
- Clawdbot (messaging-driven)
- Cursor (IDE-embedded)
- Any future agent runtime

### Loa Owns

| Concern | Description | Example |
|---------|-------------|---------|
| **Skill definitions** | What a skill does, inputs, outputs | `SKILL.md`, `index.yaml` |
| **Phase orchestration** | Order of execution, dependencies | `construct.yaml` phases |
| **Handoff validation** | What artifacts must exist between phases | `requires`, `produces` |
| **Quality criteria** | What "good" looks like | Gate definitions, rubrics |
| **Feedback schema** | What learnings to capture | `feedback/*.yaml` structure |
| **Escalation criteria** | When to ask for human help | Exit code meanings |
| **Templates** | PRD, SDD, report formats | `templates/*.md` |

### Loa Does NOT Own

- How to persist state (just says "persist this")
- How to manage tokens (just says "I need context")
- How to send messages (just says "escalate to human")
- How to recover from crashes (just says "resume from checkpoint")

### Test: Is This Loa?

> "Would this apply to ANY agent using Loa, regardless of runtime?"

If yes → Loa concern
If no → Runtime concern

---

## Runtime = How + Where (Execution Environment)

**Runtime is framework-agnostic.** It should support:
- Loa workflows
- Custom agent patterns
- Simple one-shot tasks
- Any methodology

### Runtime Owns

| Concern | Description | Example |
|---------|-------------|---------|
| **Token management** | Context limits, compaction | Soft/hard limits, estimation |
| **Memory persistence** | Where files go, how they're read | File system, database |
| **Session management** | Conversation state, history | Session IDs, compaction |
| **Tool execution** | How tools are invoked | Tool definitions, timeouts |
| **Message routing** | Telegram, Discord, etc. | Channel plugins |
| **Crash recovery** | Restart, resume | Process management |
| **Resource limits** | Cost, time, API rate limits | Quotas, throttling |
| **Authentication** | API keys, tokens | Secrets management |

### Runtime Does NOT Own

- What phases to execute (Loa tells it)
- What constitutes a valid artifact (Loa defines)
- When to escalate (Loa criteria)
- What feedback to capture (Loa schema)

### Test: Is This Runtime?

> "Would this apply to ANY workflow running on this runtime, regardless of methodology?"

If yes → Runtime concern
If no → Framework concern

---

## Integration Layer = The Contract

This is where Loa and Runtime agree to interoperate.

### Exit Codes (Loa → Runtime)

```
Loa defines meaning:
  0 = success (proceed)
  1 = failure (retry)
  2 = blocked (escalate)

Runtime implements handling:
  if exit_code == 0: proceed_to_next()
  if exit_code == 1: retry_with_limit()
  if exit_code == 2: escalate_to_human()
```

**Loa says WHAT the codes mean. Runtime says HOW to handle them.**

### State Files (Shared Schema)

```yaml
# Loa defines the schema
checkpoint:
  phase: discovery
  summary: "..."
  decisions: [...]
  
# Runtime handles persistence
# - Where to store
# - How to read/write
# - Recovery from corruption
```

**Loa says WHAT to persist. Runtime says HOW to persist.**

### Context Signals (Runtime → Loa)

```
Runtime provides signals:
  - context_tokens_available: 80000
  - time_remaining_ms: 300000
  - retry_count: 1
  
Loa uses signals to adapt:
  - if low context: summarize more aggressively
  - if low time: skip optional steps
  - if retrying: try different approach
```

**Runtime says WHAT resources are available. Loa says HOW to use them.**

### Escalation (Loa → Runtime → Human)

```
Loa triggers:
  escalate(reason: "blocked on API design", context: {...})

Runtime delivers:
  - Format for channel (Telegram, Slack, etc.)
  - Ensure delivery
  - Capture response
  - Route back to Loa
```

**Loa says WHEN and WHY to escalate. Runtime says HOW to deliver.**

---

## Practical Delineation

### When I Add a Feature, Ask:

1. **Does it define WHAT to do?** → Loa (PR to `0xHoneyJar/loa`)
2. **Does it define HOW to execute?** → Runtime (PR to `0xHoneyJar/clawdbot`)
3. **Does it connect them?** → Integration (may need PRs to both)

### Examples

| Feature | Layer | Why |
|---------|-------|-----|
| "Skills should checkpoint after each phase" | Loa | Defines WHAT to persist |
| "Checkpoint files are written to grimoires/" | Loa | Defines WHERE in project |
| "Detect when context > 80K tokens" | Runtime | HOW to measure |
| "Trigger compaction when limit exceeded" | Runtime | HOW to manage resources |
| "Checkpoint schema includes summary field" | Integration | Shared contract |
| "Exit code 2 means blocked" | Integration | Shared contract |
| "Send escalation to Telegram" | Runtime | HOW to deliver |
| "Escalation should include last 3 decisions" | Loa | WHAT context to include |

### The Context Management Protocol

From the earlier PR comment, here's how it splits:

**Loa side:**
- Checkpoint schema (what fields)
- When to checkpoint (after each phase)
- What to include in summary
- construct.yaml `context_management` config

**Runtime side:**
- Token estimation algorithm
- Compaction implementation
- File persistence
- Session state clearing
- Resume prompt generation

**Integration:**
- Checkpoint file format (shared schema)
- Soft/hard limit values (configurable)
- Restoration protocol (how Loa tells runtime to resume)

---

## Why This Matters

### For Loa (Enables All Agents)

If Loa stays runtime-agnostic:
- Works on Clawdbot today
- Works on other runtimes tomorrow
- Other teams can adopt Loa without adopting our runtime
- Learnings transfer to any agent

### For Runtime (Enables All Frameworks)

If runtime stays framework-agnostic:
- Supports Loa today
- Supports custom patterns tomorrow
- Other methodologies can run on it
- Not locked to one way of working

### For Me (Legba)

I get the benefits of both:
- Loa gives me methodology and structure
- Runtime gives me execution and resources
- I can improve either independently
- My learnings benefit both ecosystems

---

## Decision Framework

When contributing to either repo:

```
┌─────────────────────────────────────────────────────────┐
│              NEW FEATURE / IMPROVEMENT                  │
└─────────────────────────────────────────────────────────┘
                         │
                         ▼
         ┌───────────────────────────────┐
         │ Is it about WHAT to do?       │
         │ (methodology, criteria, order)│
         └───────────────────────────────┘
                    │
          ┌────────┴────────┐
          ▼                 ▼
         YES               NO
          │                 │
          ▼                 ▼
    ┌──────────┐    ┌───────────────────────────┐
    │   LOA    │    │ Is it about HOW to execute?│
    │   PR     │    │ (resources, delivery, infra)│
    └──────────┘    └───────────────────────────┘
                              │
                    ┌────────┴────────┐
                    ▼                 ▼
                   YES               NO
                    │                 │
                    ▼                 ▼
              ┌──────────┐    ┌──────────────────┐
              │ RUNTIME  │    │ INTEGRATION      │
              │   PR     │    │ (PRs to both)    │
              └──────────┘    └──────────────────┘
```

---

*This separation is what allows compound improvements. Loa gets better and all runtimes benefit. Runtime gets better and all frameworks benefit. The intersection is small and well-defined.*
