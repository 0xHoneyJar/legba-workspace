# Emerging Agentic Error Code Standards

> **Research Date**: 2026-01-31
> **Status**: No unified standard exists yet. Fragmented approaches across vendors.

---

## 1. Model Context Protocol (MCP) — Anthropic

MCP uses **JSON-RPC 2.0** error codes as its foundation.

### Standard JSON-RPC Codes (from MCP schema.ts)

```typescript
// Standard JSON-RPC error codes
export const PARSE_ERROR = -32700;
export const INVALID_REQUEST = -32600;
export const METHOD_NOT_FOUND = -32601;
export const INVALID_PARAMS = -32602;
export const INTERNAL_ERROR = -32603;

// Implementation-specific range: [-32000, -32099]
export const URL_ELICITATION_REQUIRED = -32042;  // MCP-specific
```

### JSON-RPC 2.0 Error Code Ranges

| Range | Usage |
|-------|-------|
| -32700 | Parse error (invalid JSON) |
| -32600 | Invalid request |
| -32601 | Method not found |
| -32602 | Invalid params |
| -32603 | Internal error |
| -32000 to -32099 | **Server-defined errors** (implementation-specific) |
| All other codes | **Application-defined errors** |

### MCP-Specific Extensions

MCP adds semantic meaning beyond JSON-RPC:
- Tool execution errors return in the `error` field of JSONRPCErrorResponse
- Error objects have: `code` (number), `message` (string), `data` (optional)
- Cancellation via `notifications/cancelled` with `reason` field

**Key Insight**: MCP doesn't define skill-level success/failure semantics. It only handles protocol-level errors. Tool execution results are returned as `result`, not `error`, even if the tool "failed" at the application level.

---

## 2. OpenAI Function Calling

OpenAI's approach is **unstructured**:

- No defined error codes for tool execution
- Tool results are plain JSON returned by the application
- Model interprets success/failure from the content
- No standard schema for tool outputs

**Pattern observed**: OpenAI relies on the model to interpret natural language or JSON responses. There's no protocol-level distinction between tool success and failure.

```json
// Success
{"type": "function_call_output", "call_id": "...", "output": "{\"result\": \"done\"}"}

// Failure (same structure, different content)
{"type": "function_call_output", "call_id": "...", "output": "{\"error\": \"failed\"}"}
```

**Key Insight**: OpenAI treats tool outputs as opaque. The model decides what constitutes success.

---

## 3. Google/Gemini

No public standard found for agentic error codes. Gemini function calling follows a similar pattern to OpenAI — unstructured tool responses.

---

## 4. Vercel AI SDK

Vercel's Skills SDK (announced 2026) doesn't define error codes either. Skills return:
- Structured JSON responses
- Natural language responses
- The agent interprets success/failure

---

## 5. LangChain / LangGraph

LangChain uses Python exceptions internally but no standardized error codes across tools. Tool errors typically raise exceptions that the framework catches.

---

## Comparison Table

| Framework | Error Model | Structured Codes? | Success/Failure Signal |
|-----------|-------------|-------------------|------------------------|
| **MCP** | JSON-RPC 2.0 | Yes (protocol-level) | No (application interprets) |
| **OpenAI** | Unstructured | No | Model interprets output |
| **Google** | Unstructured | No | Model interprets output |
| **Vercel** | Unstructured | No | Model interprets output |
| **LangChain** | Exceptions | No | Python exceptions |
| **BSD sysexits** | Exit codes | Yes (deprecated) | Numeric codes |

---

## Synthesis: What Should Loa Do?

### Option A: Follow MCP (JSON-RPC style)

Use negative integer error codes for protocol errors:
```yaml
-32700: parse_error
-32600: invalid_request  
-32601: method_not_found
-32602: invalid_params
-32603: internal_error
-32000 to -32099: loa_specific_errors
```

**Pros**: Aligns with MCP, familiar to JSON-RPC users
**Cons**: Negative codes are unusual for CLI tools, doesn't map to Unix exit codes

### Option B: Stay Unix (sysexits style)

Use positive integers following BSD conventions:
```yaml
0: success
1-63: application errors
64-78: sysexits standard
126-127: shell conventions
```

**Pros**: Works with shell scripts, familiar to Unix users, positive integers
**Cons**: Doesn't align with MCP, deprecated standard

### Option C: Hybrid Approach

- **Protocol layer**: JSON-RPC codes (MCP-compatible)
- **Skill execution layer**: Unix exit codes (shell-compatible)
- **Mapping**: Protocol wraps skill execution

```yaml
# Skill returns Unix exit code
skill_exit_code: 1

# Orchestrator translates to JSON-RPC for MCP consumers
{
  "error": {
    "code": -32000,  # Server error range
    "message": "Skill execution failed",
    "data": {
      "exit_code": 1,
      "category": "failure"
    }
  }
}
```

**Pros**: Best of both worlds
**Cons**: More complexity, two systems to understand

---

## Recommendation for PR #73

Given the current landscape:

1. **No dominant standard exists** — the field is fragmented
2. **MCP is the closest to a standard** for agent protocols
3. **Unix exit codes are better for CLI/shell contexts**

### Practical Approach

**For Loa skills running in Claude Code (CLI context)**:
- Use V2 minimal Unix codes: 0, 1, 2
- These work with shell scripts and are simple

**If Loa exposes an MCP server interface later**:
- Wrap skill exit codes in JSON-RPC error responses
- Map to -32000 range for MCP compatibility

**Don't invent a new standard** — wait for the ecosystem to converge.

---

## References

- MCP Specification: https://modelcontextprotocol.io/specification/2025-11-25
- MCP Schema: https://github.com/modelcontextprotocol/specification/blob/main/schema/2025-11-25/schema.ts
- JSON-RPC 2.0: https://www.jsonrpc.org/specification
- BSD sysexits: https://man.freebsd.org/cgi/man.cgi?query=sysexits
- OpenAI Function Calling: https://platform.openai.com/docs/guides/function-calling

---

*Research conducted 2026-01-31*
