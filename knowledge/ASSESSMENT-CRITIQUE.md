# Adversarial Critique: Assessment Architecture Design

> **TL;DR**: This document is a beautiful cathedral built on speculative foundations. It defines intricate interfaces for systems that don't exist, solves problems we haven't encountered, and asks skill authors to implement contracts nobody will enforce. Simplify ruthlessly or watch it rot.

---

## 1. Complexity vs Value: The Speculative Architecture Problem

### You're Building Interfaces for Ghosts

The document explicitly admits:

> "This design explicitly **does not** include:
> - An automated scoring engine
> - LLM-as-judge implementation
> - Quality metrics dashboards"

Then proceeds to define **hundreds of lines of interfaces** for those non-existent systems. This is architecture astronautics. You're defining `SkillExecutionResult` with 15+ fields, `PhaseHandoff` contracts, five assessment gates, 25+ exit codes—for a scoring engine that *might* exist *someday*.

**Counter-principle**: YAGNI (You Ain't Gonna Need It). Define interfaces when you have consumers. Not before.

### What's the Simplest Thing That Could Work?

Here it is:

```typescript
interface SkillResult {
  success: boolean;
  output: any;
  error?: string;
}
```

Exit code: 0 or 1. Log to stdout. Done.

When you actually build a scoring engine and discover you need `confidence: number | null` or `evidence: Evidence[]`, add them then. Not now. Not speculatively.

### The "Capture Now, Analyze Later" Trap

> "**Principle: Capture now, analyze later.**"
> 
> "We don't know exactly what metrics will matter for assessment."

This is a recipe for:
1. Capturing mountains of data nobody reads
2. Paying storage costs for speculation
3. Schema rot when the "later" analysis wants different data
4. Privacy liability for data you didn't need

**Better principle**: Capture minimally. Add fields when analysis proves their value.

---

## 2. Adoption Reality: The Friction Problem

### Will Skill Authors Actually Implement This?

Look at what you're asking skill authors to produce:

```typescript
interface SkillExecutionResult {
  execution_id: string;
  skill_id: string;
  skill_version: string;
  status: 'success' | 'failure' | 'partial' | 'blocked' | 'timeout';
  exit_code: number;
  started_at: ISO8601;
  completed_at: ISO8601;
  duration_ms: number;
  inputs: { provided: Record<string, any>; defaults_used: string[]; missing: string[] };
  outputs: { primary: any; artifacts: Artifact[]; side_effects: SideEffect[] };
  diagnostics: { gates_passed: GateResult[]; errors: Error[]; warnings: Warning[]; retries: RetryAttempt[] };
  assessment: { confidence: number | null; goal_statement: string | null; evidence: Evidence[]; human_review_requested: boolean };
}
```

**Realistic adoption**: Zero. Skill authors will:
- Return `{ success: true, output: "done" }`
- Ignore exit codes (everything returns 0 or 1)
- Skip artifacts, side_effects, evidence entirely
- Never populate `confidence` or `goal_statement`

Without enforcement, contracts become documentation that lies.

### The Exit Code Illusion

> "Following Unix convention and BSD sysexits.h, we define skill exit codes"

Here's the reality: **Even Unix programs don't follow sysexits.h consistently.** It's a 40-year-old standard that most software ignores. `grep` returns 1 for "no match" and 2 for errors. `diff` returns 0, 1, or 2. `curl` has its own 90+ codes.

You're defining:
```yaml
64:  usage_error
65:  data_error
66:  no_input
67:  no_user       # Addressee unknown? In an AI skill?
68:  no_host
...
```

Skill authors will use: 0 (success), 1 (failure). Maybe 2 if they're fancy. The rest is fiction.

### The 25-Exit-Code Problem

Your exit code table has 25+ codes. Quick: what's the difference between:
- `1: general_failure`
- `3: incomplete`
- `4: partial`
- `70: internal_error`
- `75: temp_fail`

In practice? Nothing. They all mean "something went wrong." Semantic distinctions without semantic enforcement are noise.

---

## 3. Edge Cases: What Breaks?

### Concurrent Execution Writes

> "Execution traces stored as append-only JSONL"

Okay, but:
- Two skills executing in parallel write to the same file?
- Append is NOT atomic on most filesystems without explicit locking
- JSONL readers assume one complete JSON object per line—what happens with interleaved partial writes?

**Not addressed**: File locking, per-execution files, rotation, or any concurrency strategy.

### Partial Pipeline Failures

```
Phase A → Handoff → Phase B → Handoff → Phase C
              ↓
        (fails validation)
```

What happens when:
- Phase A completes but handoff validation fails?
- Phase B starts, writes partial artifacts, then crashes?
- Phase C's preconditions pass but execution fails halfway through?

The document says:
> "on_failure: block       # block | warn | skip"

But doesn't specify:
- Who cleans up partial state?
- How do you resume?
- What's the rollback strategy?
- How do you handle idempotency?

### Schema Evolution

> "schema: prd-v1"
> "schema: sdd-v1"

What happens when:
- prd-v2 is introduced?
- Old traces reference v1 schemas?
- A skill produces v2 output but the handoff expects v1?

**Not addressed**: Schema versioning, migration, backward compatibility, or deprecation strategy.

### Network Issues / Distributed Execution

If skills run remotely (cloud, worker nodes), what happens when:
- Network drops during execution?
- Trace upload fails?
- Clock skew makes timestamps unreliable?
- Execution completes but result delivery fails?

**Not addressed**: Durability, at-least-once delivery, distributed tracing correlation.

---

## 4. Missing Pieces

### Who Writes the Traces? Where Do They Go?

The document says:
> "Execution traces stored as append-only JSONL"

But doesn't specify:
- **Who writes them**: The skill? An orchestrator wrapper? A framework?
- **Where they go**: Local file? Central service? Cloud storage?
- **What path**: `./traces.jsonl`? `/var/log/loa/traces/`? `s3://bucket/traces/`?
- **Naming convention**: Per-execution? Per-day? Per-skill?

### How Do You Query Them?

> "results = load_execution_traces("2026-01-*.jsonl")"

Cute pseudocode. In reality:
- JSONL files don't have indexes
- Glob-loading a month of traces could be gigabytes
- "Show me all failures for skill X" requires full scan
- Time-range queries need parsing every timestamp

**Alternatives not mentioned**: SQLite, TimescaleDB, ClickHouse, even structured log aggregators like Loki. All of which exist and are battle-tested.

### Storage Costs

Let's do math. Per execution, you're capturing:
- Execution ID, skill ID, version, timing, status (~200 bytes)
- Inputs (hashed: ~100 bytes; full: unbounded)
- Outputs summary (~500 bytes? unbounded?)
- 5 gate results with checks (~300 bytes)
- Artifacts metadata (~200 bytes)
- Side effects (~100 bytes)

**Conservative estimate**: 1-2KB per execution.

A moderately busy system doing 10,000 skill executions/day:
- 10-20MB/day
- 300-600MB/month
- 4-7GB/year

**If you capture full inputs/outputs**: 10-100x that. Now you're at 40-700GB/year.

Who's paying for this? Who's cleaning it up? The document says "retention_days: 30" but doesn't describe the cleanup mechanism.

### Versioning When Interfaces Change

You define `SkillExecutionResult` version 1. Then you ship it. Then you realize:
- `confidence` should be `confidence_score` and be required
- `evidence` needs a `source` field you forgot
- `side_effects` should track `timestamp`

Now you have:
- Old traces with old schema
- New code expecting new schema
- Queries that break on mixed data

**Not addressed**: Interface versioning, backward compatibility guarantees, migration tooling.

---

## 5. Alternative Approaches

### Just Use HTTP Status Codes

You're building a skill execution framework. Skills are essentially RPC calls. HTTP figured this out:

| HTTP | Meaning | Your Equivalent |
|------|---------|-----------------|
| 200 | Success | exit 0 |
| 400 | Bad Request | exit 64-66 |
| 401/403 | Auth | exit 77 |
| 404 | Not Found | exit 127 |
| 408 | Timeout | exit 81 |
| 500 | Internal Error | exit 70 |
| 503 | Unavailable | exit 69, 75 |

HTTP has 3 decades of tooling, middleware, and developer familiarity. Your custom exit codes have... this document.

### Just Use OpenTelemetry

> "Execution traces stored as append-only JSONL"

OpenTelemetry exists. It provides:
- Standardized trace format
- Distributed trace correlation
- Automatic instrumentation
- Exporters to every observability platform
- Battle-tested at Google/Facebook scale

Your custom JSONL format provides:
- Another standard to maintain
- No ecosystem
- No tooling
- Bespoke everything

### Just Use Structured Logging

Instead of a bespoke `SkillExecutionResult` schema, emit structured logs:

```json
{"level": "info", "event": "skill_start", "skill": "discovery", "execution_id": "abc123"}
{"level": "info", "event": "skill_complete", "skill": "discovery", "status": "success", "duration_ms": 1234}
```

Then use Loki, Elasticsearch, or CloudWatch Logs Insights to query. These systems exist. They scale. They have dashboards.

### Just Use Database Events

Instead of JSONL files:

```sql
CREATE TABLE skill_executions (
  id UUID PRIMARY KEY,
  skill_id TEXT,
  status TEXT,
  exit_code INT,
  started_at TIMESTAMPTZ,
  completed_at TIMESTAMPTZ,
  inputs JSONB,
  outputs JSONB
);
```

Now you have:
- Indexes
- SQL queries
- Transactions
- Constraints
- Backup tooling
- 50 years of database wisdom

---

## 6. Specific Quotes That Concern Me

### "Capture now, analyze later"

This is how you end up with data swamps. Real principle: **Capture what you need. Add what you learn you need.**

### "Without structured assessment, autonomous agents operate as black boxes"

They already do. Adding interfaces doesn't change that. Implementation does. This document is interfaces without implementation.

### "We adapt this for AI skill execution"

Unix exit codes work because:
1. Programs are deterministic
2. Inputs are well-defined
3. Outputs are predictable
4. Errors are enumerable

AI skill execution is:
1. Non-deterministic
2. Inputs are fuzzy (natural language)
3. Outputs vary wildly
4. "Failure" is subjective

The Unix analogy is seductive but misleading.

### "Five assessment gates"

> Gate 0 | **Selection** | Was the correct skill chosen? | Medium difficulty
> Gate 4 | **Goal Achievement** | Did it achieve the objective? | Hard difficulty

You've defined two "hard" assessment gates (requiring LLM-as-judge or human review) and three easy ones (automated checks). But the hard ones are **the only ones that matter** for quality assessment. 

Gates 1-3 (precondition, execution, output validation) are just "did it run without crashing and produce valid JSON." That's table stakes, not quality.

---

## 7. Constructive Suggestions

### 1. Start With One Thing

Don't define 5 gates, 25 exit codes, 15-field result objects, and handoff protocols simultaneously. Pick ONE:

- **Option A**: Exit codes only. Implement them. Prove they're useful.
- **Option B**: JSONL traces only. Capture minimally. Add fields when needed.
- **Option C**: Handoff validation only. Make phase transitions reliable.

### 2. Use Existing Standards

- Traces: OpenTelemetry
- Logs: Structured logging to existing aggregators
- Storage: SQLite or Postgres, not bespoke JSONL
- Status codes: HTTP semantics (they're already understood)

### 3. Define Enforcement, Not Just Interfaces

Every interface should answer:
- Who produces this data?
- Who consumes it?
- What happens if it's missing/invalid?
- Who's responsible for enforcement?

### 4. Version Everything From Day 1

```typescript
interface SkillExecutionResult {
  schema_version: "1.0.0";  // ALWAYS include this
  // ...
}
```

### 5. Build the Scoring Engine First

Seriously. Build a minimal scoring engine that works on *current* execution output (whatever that is). Then discover what data you actually need. Then define interfaces to capture it.

Design backward from real consumers, not forward from imagined ones.

---

## Summary

This document exhibits **Speculative Architecture Syndrome**: elaborate designs for systems that don't exist, solving problems that haven't been encountered, with interfaces no one will implement.

The fix isn't iteration—it's reduction. Cut 80% of this. Keep:
1. Exit code 0 (success) and 1 (failure)
2. A simple result object with 5 fields
3. Structured logging to stdout
4. A promise to add more when you need more

Everything else is premature optimization of a system that doesn't exist yet.

---

*Critique by: Skeptical Systems Architect*
*Date: 2025-01-31*
*Verdict: Overly ambitious. Needs ruthless simplification.*
