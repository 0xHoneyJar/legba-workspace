# Assessment Design Review Process

> **Meta-learning document**: Tracking how we iterate on designs, what worked, what didn't.

---

## Review Process Used

### Phase 1: Rubric-Based Self-Review
**Approach**: Systematic critique against specific questions before external review.

### Phase 2: Adversarial Sub-Agent Critique  
**Approach**: Spawn fresh session to attack the design from different angles.

### Phase 3: Synthesis & Iteration
**Approach**: Merge feedback, update design, document learnings.

---

## Phase 1: Rubric Self-Review

**Date**: 2026-01-31  
**Reviewer**: Legba (author)  
**Bias warning**: Author reviewing own work — likely to miss blind spots.

### Rubric Questions

#### 1. Does every interface have a clear owner?

| Interface | Owner | Verdict |
|-----------|-------|---------|
| SkillExecutionResult | Skill produces, orchestrator consumes | ✅ Clear |
| PhaseHandoff | Sending phase produces, receiving phase validates | ✅ Clear |
| GateResult | Gate checker produces, orchestrator logs | ✅ Clear |
| Exit codes | Skill sets, orchestrator interprets | ✅ Clear |
| JSONL traces | Orchestrator writes, scorer reads | ⚠️ Who manages retention/cleanup? |

**Gap found**: Trace lifecycle management not specified. Who deletes old traces? Cron job? Manual? Framework responsibility?

#### 2. What's the simplest path that could work?

Current design has:
- 5 gates
- 20+ exit codes
- Complex TypeScript interfaces
- JSONL trace storage
- Handoff validation

**Minimal viable version**:
- Gate 2 only (execution: did it error?)
- Exit codes 0/1/2 only (success/failure/blocked)
- Result: `{status, exit_code, error_message}`
- No traces initially

**Question**: Are we over-engineering for Phase 1? The design doc says "groundwork not implementation" but the interfaces are already complex.

**Decision**: Keep full design as north star, but mark which parts are "Phase 1 essential" vs "Phase 2+ additions".

#### 3. What happens when things fail at each gate?

| Gate | Failure Mode | Current Handling | Gap? |
|------|--------------|------------------|------|
| 0 (Selection) | Wrong skill chosen | Not specified | ⚠️ How do we even detect this? |
| 1 (Precondition) | Missing inputs | Block execution | ✅ Clear |
| 2 (Execution) | Runtime error | Capture exit code, maybe retry | ✅ Clear |
| 3 (Output) | Schema mismatch | Log validation error | ⚠️ Then what? Retry? Fail? |
| 4 (Goal) | Didn't achieve intent | ??? | ⚠️ Completely unspecified |

**Gaps found**:
- Gate 0 failure detection is hand-wavy ("LLM-as-judge" but no mechanism)
- Gate 3 failure recovery path unclear
- Gate 4 is acknowledged as "hard" but no concrete approach

#### 4. Are we capturing too much? Too little?

**Too much risks**:
- Token/storage costs for traces
- Privacy concerns (inputs may contain sensitive data)
- Analysis paralysis ("we have all this data, what do we do with it?")

**Too little risks**:
- Can't diagnose failures retroactively
- Can't train scoring models
- No evidence for goal achievement claims

**Current stance**: "Capture now, analyze later" with hashed inputs by default.

**Assessment**: Reasonable balance, but should add explicit cost/storage estimates.

#### 5. What assumptions might not hold?

| Assumption | Risk Level | Mitigation |
|------------|------------|------------|
| Skills will implement the result interface | High | They won't unless forced |
| Exit codes will be used consistently | Medium | Document clearly, lint/validate |
| JSONL is good enough for traces | Low | Can migrate later if needed |
| Gates 0 and 4 can be assessed by LLM | Medium | Needs real testing |
| Handoff validation adds value vs friction | Medium | Start optional, make required later |

**Biggest risk**: Skills won't adopt the interface unless there's enforcement or clear benefit. Need adoption strategy.

---

### Phase 1 Summary

**Issues Found**:
1. Trace lifecycle management unspecified
2. Design might be over-engineered for Phase 1
3. Gate 0, 3, 4 failure handling unclear
4. No adoption strategy for skill authors
5. Missing cost/storage estimates

**Severity**: None are blockers, but #4 (adoption) is the real risk. A beautiful interface nobody uses is worthless.

---

## Phase 2: Adversarial Sub-Agent Critique

**Completed**: 2026-01-31  
**Reviewer**: Fresh Claude instance as "Skeptical Systems Architect"  
**Output**: `/root/clawd/knowledge/ASSESSMENT-CRITIQUE.md`

### Core Diagnosis: Speculative Architecture Syndrome

The design defines elaborate interfaces for systems that don't exist yet. Key problems:

1. **YAGNI violation**: 25+ exit codes, 15-field result objects, 5 assessment gates... for a scoring engine that "explicitly does not" exist
2. **Adoption fantasy**: Skill authors will ignore 90% without enforcement
3. **Edge cases unaddressed**: Concurrent writes, partial failures, schema evolution, distributed execution
4. **Reinventing wheels**: OpenTelemetry, structured logging, HTTP status codes all exist
5. **Unix analogy breaks**: AI execution is non-deterministic with fuzzy inputs—not like `grep`

### Specific Issues Raised

| Issue | Severity | Valid? |
|-------|----------|--------|
| Exit codes beyond 0/1 won't be used | High | Yes — even Unix programs ignore sysexits |
| Nobody will populate 15 result fields | High | Yes — without enforcement, contracts lie |
| JSONL has concurrency issues | Medium | Yes — no locking strategy specified |
| "Capture now, analyze later" → data swamp | Medium | Partially — but some capture is needed |
| Should use OpenTelemetry instead | Medium | Maybe — depends on deployment context |
| Gates 0 and 4 are "hard" but they're the only ones that matter | High | Yes — insightful point |

### The Killer Quote

> "The fix isn't iteration—it's reduction. Cut 80%."

---

## Phase 3: Synthesis & Iteration

### Agreements Across Both Reviews

1. **Adoption is the real risk** — Both reviews flagged this
2. **Over-engineering for Phase 1** — Self-review suspected, adversarial confirmed
3. **Trace lifecycle unclear** — Both identified
4. **Gate failure handling vague** — Both identified

### Where I Disagree With The Critique

1. **"Just use OpenTelemetry"** — Valid for tracing, but doesn't solve skill-specific semantics (handoffs, phases). It's a tool, not a replacement for domain modeling.

2. **"Exit codes are fiction"** — Partially true, but having a semantic framework beats nothing. Even if skills only use 0/1/2, the orchestrator can standardize interpretation.

3. **"Build scoring engine first"** — Chicken-and-egg. You need some interface to capture data for the scoring engine to consume. But point taken: start minimal.

### The Real Lesson

**The design was written backward.** I defined interfaces for imagined consumers instead of starting from concrete needs.

Correct approach:
1. What does the orchestrator NEED to make decisions today?
2. Define that interface and nothing more
3. When a real consumer needs more, add it then

### Proposed Minimal Viable Interface

```typescript
// V1: What we actually need for PR #73
interface SkillResult {
  status: 'success' | 'failure' | 'blocked';
  exit_code: 0 | 1 | 2;  // success | failure | blocked
  error?: string;
  outputs?: Record<string, string>;  // artifact paths
}
```

Exit codes:
- `0` — Success
- `1` — Failure (retriable)  
- `2` — Blocked (needs human/escalation)

That's it. Everything else is Phase 2+.

### What To Keep From Original Design

1. **The gate model as conceptual framework** — Useful for thinking, not for interfaces
2. **Handoff concept** — But simplify to "did previous phase produce expected files?"
3. **"Capture for future scoring" principle** — But start with structured logging, not bespoke JSONL

### What To Cut

1. 25 exit codes → 3
2. 15-field result object → 5 fields
3. Bespoke JSONL traces → Structured logging (stdout)
4. Gate enforcement → Manual/review-based for now
5. PhaseHandoff type → Simple file existence check

---

## Revised Approach for PR #73

### Keep (Conceptual)
- Assessment gates as mental model (document only, not interfaces)
- Unix philosophy framing (simple contracts)
- "Capture now, analyze later" but with minimal capture

### Add (Minimal Implementation)
- 3 exit codes: 0/1/2
- 5-field SkillResult
- Structured logging to stdout
- File-based handoff validation (does artifact exist? yes/no)

### Defer (Phase 2+)
- Complex result interfaces
- JSONL trace storage
- Gate enforcement
- LLM-as-judge integration
- Everything else

---

## Action Items

1. [ ] Write ASSESSMENT-DESIGN-V2.md with minimal approach
2. [ ] Update PR #73 description to reflect simpler scope
3. [ ] Add comment acknowledging the pivot
4. [ ] Keep original design as "north star" reference, clearly marked as aspirational

---

## Meta-Learnings

### What Worked
- Rubric questions forced systematic thinking
- Writing down assumptions made them visible
- "Simplest path" question revealed over-engineering risk
- **Adversarial sub-agent was brutal but invaluable** — caught things I couldn't see
- **Two-pass review** (self then external) layered different perspectives

### What To Improve
- Should have done rubric review BEFORE writing 22KB design doc
- Need external critique earlier in process
- Consider "5 Whys" or other techniques
- **Start from concrete consumers, not imagined ones**
- **"What's the simplest thing that could work?" should be question #1, not #2**

### Process Improvements for Next Time
1. Start with: "Who consumes this? What do they need TODAY?"
2. Write 1-page design, get critique, THEN expand if justified
3. Include adoption/rollout strategy from the start
4. Estimate costs/overhead explicitly
5. **Run adversarial critique before publishing, not after**
6. **When you hear yourself say "capture now, analyze later" — stop and question it**

### The Core Lesson

**Design backward from real consumers, not forward from imagined interfaces.**

I spent hours defining `SkillExecutionResult` with 15 fields for a scoring engine that doesn't exist. If I'd asked "what does the orchestrator need to make a retry decision today?" the answer is: `{ success: bool, error?: string }`. Everything else is speculation.

### Quotes To Remember

From the critique:
> "The fix isn't iteration—it's reduction."

> "Without enforcement, contracts become documentation that lies."

> "Build the scoring engine first. Then discover what data you actually need."

---

*Document version: 1.1*
*Created: 2026-01-31*
*Updated: 2026-01-31 (post-critique synthesis)*
