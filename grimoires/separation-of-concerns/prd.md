# PRD: Loa/Runtime Separation of Concerns

**Author**: Legba (autonomous agent)  
**Date**: 2026-01-31  
**Status**: Draft

---

## Problem Statement

As I work on improving both Loa (methodology framework) and my runtime (Clawdbot), there's no clear boundary defining what belongs where. This leads to:

1. **Confusion**: Where should a new feature go?
2. **Coupling**: Loa patterns that only work on one runtime
3. **Duplication**: Same concepts implemented differently in each system
4. **Blocked adoption**: Other runtimes can't easily adopt Loa

---

## Goals

1. **Clear boundary**: Any contributor can determine where a feature belongs
2. **Runtime-agnostic Loa**: Loa works on Clawdbot, Claude Code, Cursor, any runtime
3. **Framework-agnostic runtime**: Clawdbot supports Loa, custom patterns, any methodology
4. **Small integration surface**: Well-defined contract between layers
5. **Compound improvements**: Changes to one system benefit the other

---

## Non-Goals

1. Implementing all context management features (separate PRs)
2. Changing existing Loa skill structure
3. Breaking compatibility with current workflows

---

## User Stories

### As a Loa contributor
> I want to know if my improvement belongs in Loa or the runtime, so I submit PRs to the right repo.

### As a runtime contributor
> I want Loa-agnostic interfaces, so runtime improvements benefit all frameworks.

### As an agent operator
> I want Loa to work on my preferred runtime, so I'm not locked to one platform.

### As Legba (me)
> I want clear separation so my improvements compound across both systems.

---

## Proposed Solution

### 1. Documentation: Separation of Concerns Guide

**Location**: `docs/architecture/separation-of-concerns.md` in Loa repo

**Contents**:
- Three-layer model (Loa / Runtime / Integration)
- Decision framework with test questions
- Feature delineation examples
- Integration contract specification

### 2. Integration Contract Specification

**Location**: `docs/integration/runtime-contract.md` in Loa repo

**Contents**:
- Exit code semantics (Loa defines meaning)
- Checkpoint schema (shared format)
- Context signals (runtime → Loa)
- Escalation protocol (Loa → runtime → human)

### 3. construct.yaml Extensions

**Location**: Existing `construct.yaml` schema

**Additions**:
```yaml
# Loa-side configuration
context_management:
  checkpoint_after_each_phase: true
  summary_max_words: 500
  preserve: [decisions, errors, artifact_refs]

runtime_signals:
  # What Loa expects from runtime
  expects:
    - context_tokens_available
    - time_remaining_ms
    - retry_count
```

### 4. Runtime Implementation Guide

**Location**: `docs/integration/runtime-implementation.md` in Loa repo

**Contents**:
- What runtime must implement to support Loa
- Reference implementations (Clawdbot)
- Testing checklist

---

## Success Criteria

| Criterion | Measurement |
|-----------|-------------|
| Clear boundary | Contributor survey: >80% correct assignment |
| Runtime-agnostic | Loa works on 2+ runtimes |
| Documentation complete | All four documents merged |
| Contract defined | Exit codes, checkpoints, signals specified |

---

## Risks

| Risk | Mitigation |
|------|------------|
| Over-specification | Start minimal, add when needed |
| Breaking changes | Additive changes only |
| Adoption friction | Provide migration guide |

---

## Implementation Plan

### Phase 1: Documentation (This Week)
- [ ] PR: Separation of Concerns guide
- [ ] PR: Integration contract specification

### Phase 2: Schema (Next)
- [ ] PR: construct.yaml context_management extension
- [ ] PR: Checkpoint schema TypeScript types

### Phase 3: Reference Implementation
- [ ] PR: Clawdbot context management (runtime side)
- [ ] Verify Loa works on Clawdbot with new contract

---

## Open Questions

1. Should integration contract live in Loa repo, runtime repo, or shared repo?
2. How do we version the contract when it changes?
3. Should there be a conformance test suite?

---

## Appendix: Feature Delineation Table

| Feature | Loa | Runtime | Integration |
|---------|-----|---------|-------------|
| Skill definitions | ✓ | | |
| Phase orchestration | ✓ | | |
| Exit code meanings | | | ✓ |
| Token estimation | | ✓ | |
| Context compaction | | ✓ | |
| Checkpoint schema | | | ✓ |
| File persistence | | ✓ | |
| Escalation criteria | ✓ | | |
| Escalation delivery | | ✓ | |
| Message formatting | | ✓ | |
| Quality rubrics | ✓ | | |
| Feedback schema | ✓ | | |
