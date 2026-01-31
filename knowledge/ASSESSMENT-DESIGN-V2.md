# Assessment Architecture V2 — Minimal Viable Design

> **Context**: V1 was over-engineered. This version cuts 80% and focuses on what PR #73 actually needs.

---

## Design Principles (Revised)

1. **Start minimal, add when needed** — No speculative interfaces
2. **Design from consumers** — What does the orchestrator need TODAY?
3. **Use existing tools** — Structured logging > bespoke JSONL
4. **Enforcement or nothing** — Don't define contracts nobody will follow

---

## What The Orchestrator Needs Today

For PR #73 (autonomous agent skill), the orchestrator must:

1. **Know if a phase succeeded or failed** → exit code
2. **Know why it failed** → error message
3. **Know if it should retry, escalate, or proceed** → status category
4. **Know what artifacts were produced** → file paths

That's it. Everything else is future scope.

---

## The Interface

```typescript
interface SkillResult {
  status: 'success' | 'failure' | 'blocked';
  exit_code: 0 | 1 | 2;
  error?: string;
  artifacts?: string[];  // paths to produced files
  duration_ms?: number;  // optional, for monitoring
}
```

**5 fields. Not 15.**

---

## Exit Codes

| Code | Status | Meaning | Orchestrator Action |
|------|--------|---------|---------------------|
| 0 | success | Phase completed | Proceed to next phase |
| 1 | failure | Phase failed, retriable | Retry (up to max), then escalate |
| 2 | blocked | Needs human intervention | Escalate immediately |

**3 codes. Not 25.**

Why these three?
- Every decision the orchestrator makes maps to one of: proceed / retry / escalate
- More granular codes add complexity without changing decisions
- If we need more later, we'll add them when a real consumer needs them

---

## Handoff Validation

Between phases, validate:

```yaml
# In construct.yaml
handoffs:
  discovery_to_design:
    requires:
      - grimoires/{project}/prd.md
    on_missing: block  # or: warn, skip
```

Implementation: **File existence check.** That's it.

No schema validation. No completeness scores. No min_completeness: 0.8. Just: does the file exist?

Why?
- Schema validation requires schemas (which don't exist yet)
- Completeness scoring requires a scorer (which doesn't exist yet)
- File existence is checkable TODAY with zero infrastructure

Add complexity when we have evidence it's needed.

---

## Logging (Not Tracing)

Instead of bespoke JSONL traces, use structured logging:

```bash
# Skill outputs to stdout
{"event": "phase_start", "phase": "discovery", "ts": "2026-01-31T01:00:00Z"}
{"event": "phase_complete", "phase": "discovery", "status": "success", "duration_ms": 45000}
{"event": "artifact_created", "path": "grimoires/project/prd.md"}
```

Benefits:
- Works with any log aggregator (CloudWatch, Loki, even `grep`)
- No custom storage infrastructure
- Standard tooling for querying
- Zero adoption friction — skills just `console.log`

---

## Assessment Gates — Conceptual Only

Keep the 5-gate model as a **thinking tool**, not an interface:

| Gate | Question | How We Check (V2) |
|------|----------|-------------------|
| 0 - Selection | Right skill? | Human review (no automation) |
| 1 - Precondition | Ready to run? | File existence check |
| 2 - Execution | Ran without errors? | Exit code |
| 3 - Output | Valid artifacts? | File existence check |
| 4 - Goal | Achieved intent? | Human review (no automation) |

**Note**: Gates 0 and 4 are the only ones that matter for quality. Gates 1-3 are table stakes. We automate the easy ones, leave the hard ones to humans for now.

---

## What construct.yaml Should Contain (V2)

```yaml
# construct.yaml — minimal version
apiVersion: loa/v1
kind: Construct
metadata:
  name: autonomous-agent
  version: 1.0.0

phases:
  - name: discovery
    produces: [grimoires/{project}/prd.md]
    
  - name: design
    requires: [grimoires/{project}/prd.md]
    produces: [grimoires/{project}/sdd.md]
    
  - name: implementation
    requires: [grimoires/{project}/sdd.md]
    produces: [src/**/*]

execution:
  on_failure: retry    # retry | escalate | skip
  max_retries: 2
  
# That's it. No assessment config. No gate definitions. No trace config.
```

---

## What We're Deferring

| Feature | V1 Had It | V2 Status | When To Add |
|---------|-----------|-----------|-------------|
| 25 exit codes | ✅ | ❌ Cut | When orchestrator needs finer decisions |
| 15-field SkillResult | ✅ | ❌ Cut to 5 | When a consumer needs specific fields |
| JSONL trace storage | ✅ | ❌ Cut | When we build the scoring engine |
| Gate enforcement | ✅ | ❌ Deferred | When we have schemas to validate |
| LLM-as-judge | ✅ | ❌ Deferred | When we tackle Gate 0/4 automation |
| Schema validation | ✅ | ❌ Deferred | When schemas exist |
| Confidence scores | ✅ | ❌ Cut | When scoring engine needs them |

---

## Migration Path

If we later need V1's complexity:

1. **More exit codes**: Add 3 (partial) when retry-vs-proceed needs nuance
2. **More result fields**: Add them one at a time as consumers request
3. **Trace storage**: Add when we have a scoring engine that consumes traces
4. **Schema validation**: Add when we have schemas worth validating

The key: **add in response to real needs, not imagined ones.**

---

## Summary

| Aspect | V1 | V2 |
|--------|----|----|
| Exit codes | 25+ | 3 |
| Result fields | 15 | 5 |
| Trace storage | Custom JSONL | Structured logging |
| Gate enforcement | Defined | Deferred |
| Handoff validation | Schema + completeness | File existence |
| Lines of design doc | 600+ | ~150 |

**V2 does 20% of what V1 specified but covers 80% of actual needs.**

---

*Version: 2.0*
*Created: 2026-01-31*
*Philosophy: Start minimal, add when needed*
