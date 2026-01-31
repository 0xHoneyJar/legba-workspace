# Assessment Architecture Design

> **Purpose**: Establish the foundational interfaces, conventions, and data capture model that enables future automated assessment of skill execution quality. This document lays groundwork—not implementation—for PR #73.

---

## 1. Why Assessment Matters

### 1.1 The Problem We're Solving

When an AI agent executes a skill autonomously, we need to answer:

1. **Did it work?** (Binary: success/failure)
2. **How well did it work?** (Quality: excellent/acceptable/poor)
3. **Why did it succeed or fail?** (Diagnosis: what went wrong, where)
4. **Can we improve next time?** (Learning: patterns, friction points)

Without structured assessment, autonomous agents operate as black boxes. We get outputs but no insight into process quality, no ability to tune, no confidence metrics for escalation decisions.

### 1.2 The Unix Philosophy Foundation

Unix solved this decades ago with a simple, composable model:

```
command < input > output
echo $?  # exit code tells you what happened
```

**Exit codes are contracts.** A well-behaved Unix tool:
- Returns 0 on success
- Returns non-zero with semantic meaning on failure
- Produces structured output that downstream tools can parse
- Logs diagnostics to stderr without polluting stdout

We adapt this for AI skill execution:
- Skills return structured results with status codes
- Outputs conform to schemas downstream consumers expect
- Diagnostic/trace information captured separately from primary output
- Composition possible: one skill's output feeds another's input

### 1.3 What We're NOT Building (Yet)

This design explicitly **does not** include:
- An automated scoring engine
- LLM-as-judge implementation
- Quality metrics dashboards
- A/B testing infrastructure

We're laying pipes and defining contracts. The scoring engine comes later, but it needs these interfaces to exist.

---

## 2. Assessment Gates

Assessment can occur at multiple points during skill execution. Each gate represents a checkpoint where quality can be measured.

### 2.1 Gate Model

```
┌─────────────────────────────────────────────────────────────────────┐
│                         SKILL EXECUTION                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐     │
│  │  GATE 0  │───▶│  GATE 1  │───▶│  GATE 2  │───▶│  GATE 3  │     │
│  │ Selection│    │  Precond │    │ Execution│    │  Output  │     │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘     │
│       │               │               │               │            │
│       ▼               ▼               ▼               ▼            │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐     │
│  │ Was this │    │ Were pre-│    │ Did the  │    │ Does out-│     │
│  │ the right│    │ conditions│   │ skill run│    │ put match│     │
│  │ skill?   │    │ met?     │    │ without  │    │ expected │     │
│  │          │    │          │    │ errors?  │    │ schema?  │     │
│  └──────────┘    └──────────┘    └──────────┘    └──────────┘     │
│                                                                     │
│                              ┌──────────┐                          │
│                              │  GATE 4  │                          │
│                              │  Goal    │                          │
│                              │Achievement│                         │
│                              └──────────┘                          │
│                                   │                                │
│                                   ▼                                │
│                              ┌──────────┐                          │
│                              │ Did it   │                          │
│                              │ achieve  │                          │
│                              │ the      │                          │
│                              │ intent?  │                          │
│                              └──────────┘                          │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 Gate Definitions

| Gate | Name | Question | Assessable By | Difficulty |
|------|------|----------|---------------|------------|
| 0 | **Selection** | Was the correct skill chosen for this intent? | Human review, LLM-as-judge | Medium |
| 1 | **Precondition** | Were all prerequisites satisfied before execution? | Automated checks | Easy |
| 2 | **Execution** | Did the skill complete without runtime errors? | Exit codes, exceptions | Easy |
| 3 | **Output Validation** | Does output conform to expected schema/format? | JSON Schema, type checks | Easy |
| 4 | **Goal Achievement** | Did execution achieve the stated objective? | LLM-as-judge, human review | Hard |

**Key insight**: Gates 1-3 are automatable today with simple tooling. Gates 0 and 4 require judgment (human or LLM). Our design must support both.

### 2.3 Gate Timing

```
BEFORE EXECUTION
├── Gate 0: Selection (implicit - already happened)
├── Gate 1: Precondition check
│
DURING EXECUTION
├── Gate 2: Execution monitoring (errors, timeouts, retries)
│
AFTER EXECUTION
├── Gate 3: Output validation
├── Gate 4: Goal achievement (may be deferred/async)
```

---

## 3. Exit Code Semantics

### 3.1 Standard Exit Codes

Following Unix convention and BSD sysexits.h, we define skill exit codes:

```yaml
# Standard exit codes for Loa skills
exit_codes:
  # Success
  0:   success           # Skill completed successfully
  
  # General failures (1-63)
  1:   general_failure   # Unspecified error
  2:   misuse            # Invalid arguments or invocation
  3:   incomplete        # Skill ran but didn't finish all work
  4:   partial           # Some objectives achieved, others failed
  
  # Precondition failures (64-79)
  64:  usage_error       # Command line usage error
  65:  data_error        # Input data was incorrect
  66:  no_input          # Required input file missing
  67:  no_user           # Addressee unknown
  68:  no_host           # Host name unknown
  69:  unavailable       # Service unavailable
  70:  internal_error    # Internal software error
  71:  os_error          # System error (can't fork, etc.)
  72:  os_file           # Critical OS file missing
  73:  cant_create       # Can't create output file
  74:  io_error          # I/O error
  75:  temp_fail         # Temporary failure; retry later
  76:  protocol          # Remote protocol error
  77:  no_permission     # Permission denied
  78:  config_error      # Configuration error
  
  # Skill-specific (80-99)
  80:  blocked           # Skill blocked by dependency
  81:  timeout           # Skill exceeded time limit
  82:  quota_exceeded    # Resource quota exhausted
  83:  escalation        # Skill needs human intervention
  84:  validation_failed # Output failed schema validation
  
  # Reserved (100-125)
  # 100-125: Reserved for framework use
  
  # Special (126-127, 128+)
  126: not_executable    # Skill exists but isn't executable
  127: not_found         # Skill not found
  # 128+N: Skill terminated by signal N
```

### 3.2 Exit Code Design Principles

1. **Zero means success.** Always. No exceptions.
2. **Non-zero is informative.** The code tells you *why* it failed.
3. **Ranges have meaning.** 1-63 general, 64-79 precondition, 80-99 skill-specific.
4. **Composable with Unix tools.** `skill && next_skill` works as expected.
5. **Signals pass through.** 128+N indicates killed by signal N (SIGTERM=143, SIGKILL=137).

### 3.3 Why This Matters for Assessment

Exit codes enable automated triage:

```bash
# Pseudocode for orchestrator
case $exit_code in
  0)        record_success ;;
  1-3)      record_soft_failure; maybe_retry ;;
  64-78)    record_precondition_failure; fix_inputs ;;
  80-83)    record_blocked; escalate_or_wait ;;
  84)       record_validation_failure; review_output ;;
  126-127)  record_config_error; alert_maintainer ;;
  *)        record_unknown_failure; investigate ;;
esac
```

Without semantic exit codes, all failures look the same. With them, we can build intelligent retry logic, escalation rules, and eventually scoring models that understand *categories* of failure.

---

## 4. Execution Result Contract

### 4.1 The Result Object

Every skill execution produces a structured result, regardless of success or failure:

```typescript
interface SkillExecutionResult {
  // === IDENTIFICATION ===
  execution_id: string;        // Unique ID for this execution
  skill_id: string;            // Which skill ran
  skill_version: string;       // Version of skill
  
  // === STATUS ===
  status: 'success' | 'failure' | 'partial' | 'blocked' | 'timeout';
  exit_code: number;           // Unix-style exit code
  
  // === TIMING ===
  started_at: ISO8601;         // When execution began
  completed_at: ISO8601;       // When execution ended
  duration_ms: number;         // Total wall-clock time
  
  // === INPUTS ===
  inputs: {
    provided: Record<string, any>;   // What was given
    defaults_used: string[];         // Which defaults were applied
    missing: string[];               // Required inputs that were absent
  };
  
  // === OUTPUTS ===
  outputs: {
    primary: any;                    // Main result (schema-dependent)
    artifacts: Artifact[];           // Files created, etc.
    side_effects: SideEffect[];      // External changes made
  };
  
  // === DIAGNOSTICS ===
  diagnostics: {
    gates_passed: GateResult[];      // Which gates passed/failed
    errors: Error[];                 // Errors encountered
    warnings: Warning[];             // Non-fatal issues
    retries: RetryAttempt[];         // If retries occurred
  };
  
  // === ASSESSMENT HOOKS ===
  assessment: {
    // Data for future scoring - captured now, evaluated later
    confidence: number | null;       // Self-reported confidence (0-1)
    goal_statement: string | null;   // What was this trying to achieve?
    evidence: Evidence[];            // Proof of completion
    human_review_requested: boolean; // Skill flagged for review
  };
}
```

### 4.2 Why Capture This Much?

**Principle: Capture now, analyze later.**

We don't know exactly what metrics will matter for assessment. By capturing comprehensive execution data upfront, we enable:

1. **Retroactive analysis**: "What was the failure rate for skills run in Phase 3 last month?"
2. **Pattern detection**: "Which preconditions fail most often?"
3. **Scoring model training**: "What features correlate with human-assessed quality?"
4. **Debugging**: "Why did this specific execution fail?"

The overhead of capturing this data is minimal. The cost of *not* having it when you need it is high.

### 4.3 Artifact Schema

```typescript
interface Artifact {
  type: 'file' | 'url' | 'reference';
  path: string;              // Relative path or URL
  content_type: string;      // MIME type
  size_bytes: number;
  checksum: string;          // SHA-256
  metadata: Record<string, any>;
}
```

### 4.4 Side Effect Schema

```typescript
interface SideEffect {
  type: 'file_write' | 'api_call' | 'state_change' | 'notification';
  target: string;            // What was affected
  reversible: boolean;       // Can this be undone?
  description: string;       // Human-readable summary
}
```

### 4.5 Gate Result Schema

```typescript
interface GateResult {
  gate: 0 | 1 | 2 | 3 | 4;
  name: string;              // 'selection' | 'precondition' | etc.
  passed: boolean;
  checked_at: ISO8601;
  details: {
    checks_performed: Check[];
    failures: CheckFailure[];
  };
}

interface Check {
  name: string;
  passed: boolean;
  message: string;
}
```

---

## 5. Handoff Protocol

### 5.1 The Handoff Problem

In a multi-phase pipeline like Loa (PRD → SDD → Sprint → Implement → Audit), each phase hands off to the next. These handoffs are critical assessment points:

- **Did Phase N produce what Phase N+1 needs?**
- **Is Phase N+1 receiving clean inputs?**
- **How do we trace issues back to their source phase?**

### 5.2 Handoff Contract

```typescript
interface PhaseHandoff {
  // === PROVENANCE ===
  from_phase: string;              // e.g., "discovery"
  to_phase: string;                // e.g., "design"
  handoff_at: ISO8601;
  
  // === ARTIFACTS ===
  artifacts: {
    required: ArtifactSpec[];      // What must be present
    provided: Artifact[];          // What was actually provided
    missing: string[];             // Required but absent
    extra: string[];               // Provided but not expected
  };
  
  // === VALIDATION ===
  validation: {
    schema_valid: boolean;         // Artifacts match expected schema
    completeness: number;          // 0-1, how complete are required fields
    blockers: string[];            // Issues that prevent proceeding
    warnings: string[];            // Issues that should be noted
  };
  
  // === ACCEPTANCE ===
  accepted: boolean;               // Did receiving phase accept handoff?
  rejection_reason: string | null; // If rejected, why?
  remediation: string | null;      // Suggested fix if rejected
}
```

### 5.3 Handoff as Assessment Gate

Every handoff is implicitly a Gate 3 (Output Validation) for the sending phase and a Gate 1 (Precondition) for the receiving phase:

```
Phase A                     Handoff                      Phase B
┌────────┐                 ┌────────┐                   ┌────────┐
│        │                 │        │                   │        │
│ Execute├────────────────▶│Validate├──────────────────▶│ Start  │
│        │  Artifacts      │Handoff │  Accept/Reject    │        │
│ Gate 2 │                 │Gate 3/1│                   │ Gate 1 │
└────────┘                 └────────┘                   └────────┘
```

### 5.4 Handoff Artifact Specifications

Each phase declares what it expects to receive and what it will produce:

```yaml
# From construct.yaml
phases:
  discovery:
    receives: []   # First phase, no inputs
    produces:
      - path: grimoires/{project}/prd.md
        schema: prd-v1
        required: true
      - path: grimoires/{project}/research/*.md
        schema: research-note
        required: false
        
  design:
    receives:
      - path: grimoires/{project}/prd.md
        schema: prd-v1
        required: true
        min_completeness: 0.8
    produces:
      - path: grimoires/{project}/sdd.md
        schema: sdd-v1
        required: true
```

**Why min_completeness?** A PRD might technically exist but be mostly empty. The receiving phase can require a minimum completeness score to proceed.

---

## 6. Data Capture for Future Scoring

### 6.1 The Scoring Model We'll Eventually Want

Future scoring will likely be multi-dimensional:

| Dimension | Question | Data Needed |
|-----------|----------|-------------|
| **Correctness** | Did it produce the right output? | Expected vs. actual, schema validation |
| **Completeness** | Did it finish all required work? | Checklist completion, artifacts present |
| **Quality** | How good is the output? | LLM-as-judge scores, human ratings |
| **Efficiency** | Did it use resources well? | Time, tokens, retries, API calls |
| **Compliance** | Did it follow the process? | Gates passed, protocols followed |

### 6.2 Capture Points

We instrument execution to capture data at key points:

```
CAPTURE: skill_invoked
├── skill_id, version, operator_type
├── input_summary (hashed for privacy if needed)
├── context_size, memory_state

CAPTURE: gate_checked
├── gate_id, passed, checks_performed
├── failures_if_any, remediation_attempted

CAPTURE: execution_progress
├── phase_entered, phase_exited
├── artifacts_created, side_effects

CAPTURE: execution_complete
├── status, exit_code, duration
├── outputs_summary, errors, warnings

CAPTURE: handoff_attempted
├── from_phase, to_phase
├── artifacts_provided, validation_result
├── accepted, rejection_reason
```

### 6.3 Storage Format

Execution traces stored as append-only JSONL:

```jsonl
{"ts":"2026-01-31T00:45:00Z","event":"skill_invoked","data":{...}}
{"ts":"2026-01-31T00:45:01Z","event":"gate_checked","data":{...}}
{"ts":"2026-01-31T00:45:30Z","event":"execution_complete","data":{...}}
```

**Why JSONL?**
- Append-only is safe for concurrent writes
- Line-oriented is easy to stream/process
- JSON is parseable by everything
- Compresses well

### 6.4 Privacy Considerations

Execution traces may contain sensitive data. Design principles:

1. **Hash inputs by default** unless explicitly configured otherwise
2. **Redact secrets** (API keys, tokens) automatically
3. **User controls retention** - traces can be auto-deleted after N days
4. **Opt-in for full capture** in dev/debug mode

---

## 7. Assessment-Ready construct.yaml

### 7.1 Extended Schema

Building on the existing construct.yaml pattern, we add assessment configuration:

```yaml
# construct.yaml with assessment groundwork
apiVersion: loa/v1
kind: Construct
metadata:
  name: autonomous-agent
  version: 1.0.0
  
skills:
  - name: riding-codebase
    required: true
    phase: discovery

execution:
  type: sequential
  phases: [discovery, design, implementation, quality]
  
# === ASSESSMENT CONFIGURATION ===
assessment:
  # Enable/disable data capture
  capture:
    enabled: true
    retention_days: 30
    include_inputs: hashed    # none | hashed | full
    include_outputs: summary  # none | summary | full
    
  # Gate configuration per phase
  gates:
    discovery:
      precondition:
        checks:
          - context_files_exist
          - operator_detected
      execution:
        timeout_ms: 300000
        max_retries: 2
      output:
        schema: prd-v1
        min_completeness: 0.7
        
    design:
      precondition:
        checks:
          - prd_exists
          - prd_completeness: 0.8
      execution:
        timeout_ms: 600000
        max_retries: 1
      output:
        schema: sdd-v1
        required_sections:
          - architecture
          - data_model
          - api_contracts
          
  # Exit code overrides (if skill needs custom semantics)
  exit_codes:
    80: waiting_for_human    # Override default 'blocked'
    
  # Handoff configuration
  handoffs:
    discovery_to_design:
      required_artifacts:
        - grimoires/{project}/prd.md
      optional_artifacts:
        - grimoires/{project}/research/*.md
      validation:
        schema: prd-v1
        min_completeness: 0.8
      on_failure: block       # block | warn | skip
      
    design_to_implementation:
      required_artifacts:
        - grimoires/{project}/sdd.md
      validation:
        schema: sdd-v1
        required_sections: true
      on_failure: block

# === CONTRACTS (from #71) ===
contracts:
  discovery:
    inputs:
      - context/*.md
      - AGENTS.md (optional)
    outputs:
      - grimoires/{project}/prd.md
    exit_codes:
      0: success
      1: incomplete
      2: blocked
      83: escalation_required
      
  design:
    inputs:
      - grimoires/{project}/prd.md
    outputs:
      - grimoires/{project}/sdd.md
    exit_codes:
      0: success
      1: incomplete
      84: validation_failed
```

### 7.2 What This Enables

With this structure in place, a future orchestrator can:

1. **Validate before execution**: Check preconditions automatically
2. **Monitor during execution**: Track time, retries, errors
3. **Validate after execution**: Schema-check outputs
4. **Enforce handoffs**: Block phases if previous phase didn't deliver
5. **Capture traces**: Log everything for later analysis
6. **Support scoring**: Provide structured data for LLM-as-judge or metrics

---

## 8. Implementation Phases

### 8.1 Phase 1: Contracts & Exit Codes (PR #73)

**What we're doing now:**

- [ ] Define standard exit codes in construct.yaml
- [ ] Document exit code semantics
- [ ] Add contract definitions (inputs/outputs) per phase
- [ ] Structure for gate configuration (even if not enforced yet)

### 8.2 Phase 2: Result Capture (Future)

**Next step:**

- [ ] Implement SkillExecutionResult generation
- [ ] Add JSONL trace writing
- [ ] Capture timing, inputs (hashed), outputs (summary)
- [ ] Gate pass/fail recording

### 8.3 Phase 3: Gate Enforcement (Future)

**Building on Phase 2:**

- [ ] Precondition checks before phase start
- [ ] Output validation after phase complete
- [ ] Handoff validation between phases
- [ ] Automated retry on soft failures

### 8.4 Phase 4: Assessment Engine (Future)

**The eventual goal:**

- [ ] LLM-as-judge integration for Gate 0 and Gate 4
- [ ] Scoring models trained on captured data
- [ ] Quality dashboards
- [ ] Automated improvement suggestions

---

## 9. Summary

### What This Document Establishes

1. **A philosophy**: Unix-style exit codes and composable contracts
2. **A model**: Five assessment gates from selection to goal achievement
3. **Interfaces**: SkillExecutionResult, PhaseHandoff, GateResult
4. **Data capture**: What to record for future scoring
5. **construct.yaml extensions**: Assessment-ready configuration

### What PR #73 Should Include

From this design, PR #73 should add:

1. **Exit code definitions** in construct.yaml with semantic meaning
2. **Contract declarations** (inputs/outputs) for each phase
3. **Gate stubs** in configuration (can be unenforced placeholders)
4. **Documentation** referencing this design for future implementers

### The North Star

When someone eventually builds the assessment engine, they should be able to:

```python
# Pseudocode for future scoring
results = load_execution_traces("2026-01-*.jsonl")

for result in results:
    # Data is already there, just needs analysis
    score = assess(
        selection_gate=result.gates[0],
        precondition_gate=result.gates[1], 
        execution_gate=result.gates[2],
        output_gate=result.gates[3],
        goal_gate=llm_judge(result.goal_statement, result.evidence)
    )
    
    record_score(result.execution_id, score)
```

The pipes are laid. The contracts are defined. The data flows. Scoring is just analysis on top.

---

*Document version: 1.0*  
*Created: 2026-01-31*  
*For: PR #73 - Autonomous Agent Orchestrator*  
*Author: Legba (autonomous contribution)*
