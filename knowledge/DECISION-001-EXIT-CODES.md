# DECISION-001: Exit Code Standard for Loa Skills

**Status**: Approved  
**Date**: 2026-01-31  
**Decision Makers**: Jani (@janitooor), Legba (autonomous agent)  
**Context**: PR #73 — Autonomous Agent Orchestrator

---

## Decision

Loa skills SHALL use **Unix-style positive integer exit codes** for CLI contexts, with a minimal V2 subset for immediate implementation.

---

## V2 Minimal (Ship Now)

| Code | Meaning | Orchestrator Action |
|------|---------|---------------------|
| 0 | Success | Proceed to next phase |
| 1 | Failure (retriable) | Retry up to max, then escalate |
| 2 | Blocked | Escalate immediately |

---

## Extended Codes (Use When Needed)

If finer granularity is required, use BSD sysexits.h codes:

| Code | Constant | When to Use |
|------|----------|-------------|
| 64 | EX_USAGE | Invalid skill invocation |
| 65 | EX_DATAERR | Bad input data |
| 66 | EX_NOINPUT | Missing required input file |
| 69 | EX_UNAVAILABLE | External service down |
| 75 | EX_TEMPFAIL | Temporary failure, retry later |
| 77 | EX_NOPERM | Permission denied |
| 78 | EX_CONFIG | Configuration error |

---

## Reasoning

### Why Unix codes over JSON-RPC?

1. **Context is CLI**: Loa skills run in Claude Code, which is a shell environment. Unix exit codes work natively with `&&`, `||`, `$?`, and shell scripts.

2. **No dominant agentic standard**: Surveyed MCP, OpenAI, Google, Vercel, LangChain — no unified error code standard exists. MCP uses JSON-RPC but only for protocol errors, not skill success/failure.

3. **Simplicity**: Three codes (0/1/2) cover 95% of orchestrator decisions: proceed, retry, or escalate.

4. **BSD sysexits is deprecated but known**: It's the closest thing to a semantic exit code standard, even if imperfect.

### Why not JSON-RPC codes?

1. **Negative integers are unusual in CLI**: Shell scripts expect 0-255 range.

2. **MCP separates protocol from application**: Tool success/failure is returned in `result`, not `error`. The model interprets outcomes.

3. **Premature optimization**: Adding JSON-RPC wrapper now solves a problem we don't have yet.

### Why not invent our own?

1. **Standards should emerge from use**: The original V1 design invented codes 80-99 with no consumers. That's speculative architecture.

2. **Wait for convergence**: The agentic ecosystem is young. Better to stay minimal and adapt than lock in a non-standard approach.

---

## Future Considerations

### If Loa exposes an MCP server interface:

Wrap skill exit codes in JSON-RPC error responses:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32000,
    "message": "Skill execution failed",
    "data": {
      "skill": "discovery",
      "exit_code": 1,
      "retriable": true
    }
  }
}
```

This preserves Unix semantics while being MCP-compatible.

### If ecosystem converges on a standard:

Adopt it. This decision is not permanent — it's appropriate for current context.

---

## References

- PR #73: https://github.com/0xHoneyJar/loa/pull/73
- BSD sysexits(3): https://man.freebsd.org/cgi/man.cgi?query=sysexits
- MCP Specification: https://modelcontextprotocol.io/specification/2025-11-25
- JSON-RPC 2.0: https://www.jsonrpc.org/specification
- Research: `/root/clawd/knowledge/AGENTIC-ERROR-STANDARDS.md`

---

## Changelog

| Date | Change |
|------|--------|
| 2026-01-31 | Initial decision documented |

---

*Documented by Legba, approved by Jani*
