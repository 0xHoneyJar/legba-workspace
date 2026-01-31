# PRD Assessment: Team of Rivals Paper

**Source**: "If You Want Coherence, Orchestrate a Team of Rivals" (arXiv:2601.14351)  
**Author**: Isotopes AI (Vijayaraghavan et al.)  
**Assessed by**: Legba  
**Date**: 2026-01-31  
**Purpose**: Extract patterns for Loa (methodology) and Runtime (Clawdbot)

---

## Executive Summary

The paper proposes an "AI Office" architecture: specialized agents with defined roles, opposing incentives, and hierarchical checks. Key claim: **90% internal error interception** before user exposure.

This assessment maps extractable patterns to the Loa/Runtime separation of concerns.

---

## Pattern Extraction

### Patterns for LOA (Methodology Layer)

These belong in Loa because they define **WHAT to do** and **to what standard**.

| Pattern | Description | Loa Application |
|---------|-------------|-----------------|
| **Role Specialization** | Planners, Executors, Critics, Experts with strict boundaries | Map to Loa agent roles (discovering-requirements, architect, reviewer) |
| **Adversarial Validation** | Critics with veto power challenge executors | Formalize in `/review-sprint` gate — explicit critic persona |
| **Opposing Incentives** | Agents act as "rivals" rather than collaborators | Define in construct.yaml: critic incentive != executor incentive |
| **Quality Rubrics** | Clear criteria for what constitutes "correct" output | Already in Loa gates; can strengthen with paper's criteria |
| **Escalation Criteria** | When to ask for human help (blocked states) | Loa defines WHEN to escalate via exit codes |
| **SessionLog Lineage** | Every decision logged with traceable lineage | Formalize in grimoire structure: decisions.yaml |

**Concrete Loa Improvements:**
1. Add `critic` phase after each major output (PRD → critic, SDD → critic, code → critic)
2. Define critic success criteria: "Must challenge at least 3 assumptions"
3. Add `decisions.yaml` to grimoire schema for lineage tracking
4. Strengthen exit code 2 (blocked) with explicit escalation context

### Patterns for RUNTIME (Clawdbot Layer)

These belong in Runtime because they define **HOW to execute** with **what resources**.

| Pattern | Description | Runtime Application |
|---------|-------------|---------------------|
| **Data Hygiene** | Raw data never enters LLM context; only schemas/summaries | Context management: summarize before injecting |
| **Remote Code Executor** | Separate execution from reasoning (Jupyter-like) | Already have `exec` tool; formalize separation |
| **Graceful Degradation** | Model fallback on failure | Clawdbot already supports model fallback; strengthen |
| **Checkpoint/Resume** | Checkpointing for long operations | Implement checkpoint schema in runtime |
| **Context Isolation** | Prevent context contamination across agents | Session isolation already exists; formalize boundaries |
| **Token Management** | Keep context windows clean | Compaction, soft/hard limits |

**Concrete Runtime Improvements:**
1. Add explicit context hygiene mode: `rawData=false` for LLM calls
2. Strengthen checkpoint/resume for spawned sub-agents
3. Formalize model fallback chain in config
4. Add metrics: error interception rate, context overflow count

### Patterns for INTEGRATION (Shared Contract)

These connect Loa and Runtime.

| Pattern | Description | Integration Application |
|---------|-------------|-------------------------|
| **Error Interception Protocol** | Errors caught before user exposure | Define in contract: what counts as "caught" |
| **Hierarchical Safeguards** | Multi-layer review with escalation paths | Exit codes + escalation signals |
| **Auditability Schema** | SessionLog format for backward analysis | Shared checkpoint/lineage schema |

---

## Applicability Analysis

### High-Value Patterns (Implement Soon)

1. **Adversarial Critics in Loa**
   - Current state: Review phases exist but aren't adversarial
   - Change: Add explicit "challenge" objective to critic phases
   - Effort: Low (prompt engineering)
   - Impact: High (error reduction)

2. **Data Hygiene in Runtime**
   - Current state: Tool outputs go directly to context
   - Change: Summarize/filter before context injection
   - Effort: Medium (runtime changes)
   - Impact: High (context quality)

3. **Decision Lineage in Grimoires**
   - Current state: Decisions scattered in prose
   - Change: Add `decisions.yaml` with structured format
   - Effort: Low (schema addition)
   - Impact: Medium (auditability)

### Medium-Value Patterns (Consider Later)

4. **Error Interception Metrics**
   - Requires instrumentation of all error paths
   - Useful for measuring system health

5. **Explicit Opposing Incentives**
   - Hard to operationalize in prompts
   - Research needed on prompt patterns

### Low-Value Patterns (Skip or Deprioritize)

6. **Full SessionLog Implementation**
   - Heavy infrastructure
   - Current git history provides most value

---

## Skepticism Notes

1. **90% claim unverifiable** without their exact methodology
2. **"Team of Rivals" metaphor** may overcomplicate simple adversarial validation
3. **Zero-Human Company framing** is hype; 10% error rate still needs humans for stakes
4. **Latency tradeoffs** not quantified in abstract

---

## Proposed PRs

### For Loa Repo (0xHoneyJar/loa)

**PR 1: Adversarial Critic Protocol** ✅ DRAFTED
- Branch: `feat/adversarial-critic-protocol`
- Fork: https://github.com/janitooor/legba/pull/new/feat/adversarial-critic-protocol
- Changes:
  - Add `<adversarial_protocol>` section to reviewing-code SKILL.md
  - Require minimum 3 concerns, 1 assumption challenge, 1 alternative
  - Add adversarial analysis section to review-feedback.md template

**PR 2: Decision Lineage Schema** ✅ DRAFTED
- Branch: `feat/decision-lineage-schema`
- Fork: https://github.com/janitooor/legba/pull/new/feat/decision-lineage-schema
- Changes:
  - Add `.claude/schemas/decisions.schema.json`
  - Add `.claude/protocols/decision-capture.md`
  - Add `docs/architecture/decision-lineage.md`
  - Add `grimoires/loa/decisions.yaml` template

### For Runtime Repo (Clawdbot) — Future

**PR 3: Context Hygiene Mode** (not yet drafted)
- Add `contextHygiene: { summarize: true, maxRawBytes: 1000 }` to exec/tool config
- Summarize large outputs before context injection

**PR 4: Checkpoint Schema** (not yet drafted)
- Define shared checkpoint format for sub-agents
- Enable resume from checkpoint on failure

---

## Success Criteria

| Criterion | Measurement |
|-----------|-------------|
| Adversarial critic implemented | `/review-sprint` uses explicit challenge prompt |
| Decision lineage exists | `decisions.yaml` populated in ≥3 projects |
| Error interception measurable | Metrics dashboard shows interception rate |
| Context hygiene active | Large outputs auto-summarized |

---

## Open Questions

1. How to operationalize "opposing incentives" in prompts without artificial conflict?
2. Should critics be separate model calls or same-context personas?
3. How to measure error interception rate without ground truth?

---

## Appendix: Paper Claims vs Reality

| Claim | Assessment |
|-------|------------|
| "Reliability without scale" | Plausible — structure can substitute for capability |
| "90% error interception" | Unverifiable without methodology |
| "Maintains acceptable latency" | Vague — what's acceptable? |
| "Graceful degradation" | Standard practice, not novel |
| "Production readiness" | Marketing language |

---

*Assessment complete. Ready to proceed with PR planning.*
