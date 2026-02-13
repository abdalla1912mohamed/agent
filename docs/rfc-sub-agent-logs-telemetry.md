# RFC: Sub-Agent Logs and Telemetry

**Status:** Draft  
**Scope:** Stakpak Agent (CLI) — telemetry, logging, and observability  
**Created:** 2025-02-13

---

## 1. Summary

This RFC proposes enhancements to telemetry and logging in the **Stakpak Agent (CLI)** repository so that:

1. **Session ID** is propagated into OTLP spans for trace correlation
2. **Tool calls** are correctly attributed in traces (already implemented — documented here)
3. **Metadata** (user.id, user.name, session.id, gen_ai.capability.name) is sent to all telemetry backends via StakAI spans
4. **CLI** sends `x-session-id` / `X-Session-Id` for API correlation when using Stakpak API
5. **Cost** is out of scope for this repo (handled by server/API when applicable)

---

## 2. Key Difference from Server RFC

The original RFC was written for a **server repo** (studio/server, stakpak-ws) with Langfuse SDK integration. This agent repo is different:

| Aspect | Server Repo | Agent Repo (This) |
|--------|-------------|-------------------|
| Tracing backend | Langfuse SDK + OTLP | OpenTelemetry (OTLP) only |
| Langfuse | Direct ingestion API | Via OTLP if configured |
| Tool calls | Proposed: add Langfuse spans per tool | **Already in spans** as `gen_ai.input.messages`, `gen_ai.output.messages`, `gen_ai.tool.definitions` |
| Cost tracking | Proposed: send cost to Langfuse | N/A — no cost logic here |
| Session creation | Server creates sessions | CLI uses API session or local checkpoint |

---

## 3. Current State (Agent Repo)

### 3.1 Workspace Structure

```
cli/                    # Main binary (stakpak)
├── commands/agent/run/ # mode_interactive, mode_async, stream, tooling
libs/
├── ai/                 # StakAI (stakai) — LLM + tracing
├── api/                # AgentClient, provider, hooks
├── shared/             # stakai_adapter, LLM types
└── mcp/
```

### 3.2 Tracing and Tool Calls

| Component | File | Behavior |
|-----------|------|----------|
| **GenAI spans** | `libs/ai/src/client/mod.rs` | `generate()` and `stream()` create spans with `gen_ai.*` attributes when `tracing` feature is enabled |
| **Tool calls in spans** | `libs/ai/src/tracing.rs` | `record_input_messages()` includes `tool_call` and `tool_call_response` in `gen_ai.input.messages` JSON |
| **Tool calls in output** | `libs/ai/src/tracing.rs` | `record_streamed_response()` / `record_response_content()` include tool calls in `gen_ai.output.messages` JSON |
| **Tool definitions** | `libs/ai/src/tracing.rs` | `record_tool_definitions()` records `gen_ai.tool.definitions` |
| **Stream accumulation** | `libs/ai/src/types/stream.rs` | `accumulated_tool_calls` passed to `record_streamed_response` on `Finish` event |

**Conclusion:** Tool calls **are** included in OTLP traces. They appear as span attributes (`gen_ai.input.messages`, `gen_ai.output.messages`, `gen_ai.tool.definitions`), not as separate child spans. When Langfuse ingests OTLP, these attributes are visible in the GenAI span.

### 3.3 Metadata Flow (Current)

| Path | Metadata | Status |
|------|----------|--------|
| **StakAI telemetry_metadata** | `user.id`, `user.name`, `session.id` | **Not populated** — `stakai_adapter` always passes `telemetry_metadata: None` |
| **Session ID header** | `X-Session-Id` | Injected in `libs/api/src/client/provider.rs` when calling Stakpak API |
| **LLMInput / LLMStreamInput** | No `telemetry_metadata` field | Types do not carry metadata for spans |

### 3.4 Session ID Creation and Flow

| Location | Type | File | Notes |
|----------|------|------|-------|
| **API (remote)** | Server-created | Stakpak API | Session returned in response metadata |
| **Local** | Checkpoint/session | `libs/api/src/storage.rs` | `create_session()` etc. |
| **CLI** | Propagated | `cli/src/commands/agent/run/mode_interactive.rs` | `current_session_id` tracked, passed to provider |
| **Provider** | Injected | `libs/api/src/client/provider.rs` | `ctx.session_id` → headers, but **not** → telemetry_metadata |
| **Tool execution** | Passed | `cli/src/commands/agent/run/tooling.rs` | `session_id` in tool context for MCP/server tools |

### 3.5 LLM Request Flow

```
mode_interactive / mode_async / acp server
    → AgentClient.chat_completion_stream() / chat_completion()
    → run_agent_completion()
    → stakai.chat_stream() / stakai.chat()
    → StakAIClient (stakai_adapter)
    → GenerateRequest { telemetry_metadata: None }   ← GAP
    → Inference.generate() / stream()
```

The `GenerateRequest` is built in `libs/shared/src/models/stakai_adapter.rs`. `LLMInput` and `LLMStreamInput` do not have a `telemetry_metadata` field, so the adapter has nothing to pass through.

---

## 4. Proposed Changes

### 4.1 Task 1: Propagate Telemetry Metadata to StakAI Spans

**Objective:** Populate `user.id`, `user.name`, `session.id` (and optionally `gen_ai.capability.name`, `gen_ai.step.name`) so they appear as OTLP span attributes.

**Files:**

| File | Change |
|------|--------|
| `libs/shared/src/models/llm.rs` | Add `telemetry_metadata: Option<HashMap<String, String>>` to `LLMInput` and `LLMStreamInput` |
| `libs/shared/src/models/stakai_adapter.rs` | Use `input.telemetry_metadata` when building `GenerateRequest` instead of `None` |
| `libs/api/src/client/provider.rs` | Build metadata map from `ctx.session_id`, `ctx.state.metadata`, and user info (when available); pass to `LLMInput` / `LLMStreamInput` |

**Details:**

1. **LLMInput / LLMStreamInput** — Add optional field:
   ```rust
   pub telemetry_metadata: Option<std::collections::HashMap<String, String>>,
   ```

2. **stakai_adapter** — When building `GenerateRequest`:
   ```rust
   telemetry_metadata: input.telemetry_metadata.clone(),
   ```
   (and in `chat_stream` equivalent)

3. **provider.rs** — In `run_agent_completion`, before building `LLMStreamInput` / `LLMInput`:
   ```rust
   let mut telemetry_metadata = std::collections::HashMap::new();
   if let Some(sid) = ctx.session_id {
       telemetry_metadata.insert("session.id".to_string(), sid.to_string());
   }
   if let Some(meta) = &ctx.state.metadata {
       // Extract user.id, user.name, gen_ai.capability.name, gen_ai.step.name if present
       if let Some(v) = meta.get("user_id").and_then(|v| v.as_str()) {
           telemetry_metadata.insert("user.id".to_string(), v.to_string());
       }
       if let Some(v) = meta.get("user_name").and_then(|v| v.as_str()) {
           telemetry_metadata.insert("user.name".to_string(), v.to_string());
       }
       // ... other optional fields
   }
   input.telemetry_metadata = if telemetry_metadata.is_empty() { None } else { Some(telemetry_metadata) };
   ```

**Scope:** Add metadata plumbing; ensure `session.id` is always included when `ctx.session_id` is set.

---

### 4.2 Task 2: Ensure User and Capability Metadata Reach Context

**Objective:** Allow callers (mode_interactive, mode_async, ACP) to supply user and capability metadata that flows into spans.

**Files:**

| File | Change |
|------|--------|
| `cli/src/commands/agent/run/mode_interactive.rs` | Pass `user_id`, `user_name` (from config/account) and `gen_ai.capability.name` (e.g. `"agent_interactive"`) in metadata when calling `chat_completion_stream` |
| `cli/src/commands/agent/run/mode_async.rs` | Same for async mode |
| `cli/src/commands/acp/server.rs` | Pass Zed/user context and `gen_ai.capability.name` (e.g. `"acp"`) when calling provider |

**Metadata convention:**

- `session.id` — UUID string
- `user.id` — Account ID or anonymous ID
- `user.name` — Username or "local"
- `gen_ai.capability.name` — `"agent_interactive"`, `"agent_async"`, `"acp"`, etc.
- `gen_ai.step.name` — Optional step identifier

**Scope:** Populate metadata at entry points; Task 1 consumes it.

---

### 4.3 Task 3: Session ID Header for API Correlation

**Objective:** Ensure the CLI sends `x-session-id` or `X-Session-Id` for requests that hit the Stakpak API, so server-side traces can correlate.

**Current:** `provider.rs` injects `X-Session-Id` into `input.headers` when `ctx.session_id` is set. Headers are passed to Stakpak API calls. Verify this is applied for all relevant request paths (agent completion, checkpoint, etc.).

**Files:**

| File | Verification |
|------|--------------|
| `libs/api/src/client/provider.rs` | Already injects `X-Session-Id` in `run_agent_completion` |
| `libs/api/src/stakpak/` | Ensure API client forwards headers where session correlation is needed |

**Scope:** Documentation + verification; likely no code change.

---

### 4.4 Task 4: Tool Execution Spans (Optional)

**Objective:** Add **child spans** for each tool execution so tools appear as separate spans in traces, not only as attributes on the GenAI span.

**Current:** Tool calls are recorded inside `gen_ai.input.messages` and `gen_ai.output.messages`. There are no dedicated spans around tool execution.

**Proposed:** In `cli/src/commands/agent/run/tooling.rs` (and any path that executes tools), wrap tool execution in a span:

```rust
let span = tracing::info_span!(
    "tool.execute",
    "gen_ai.tool.name" = %tool_name,
    "gen_ai.tool.call_id" = %tool_call_id,
);
let _guard = span.enter();
// ... execute tool ...
// Result can be recorded on span if desired
```

This creates child spans under the agent’s trace. When using OTLP + Langfuse, tools would appear as distinct spans.

**Scope:** Optional enhancement; can be deferred.

---

## 5. Concrete Span Attributes and Tool Call Examples

### 5.1 Observed Span Metadata (Current)

The following attributes are present on GenAI spans when telemetry is configured (e.g. via Stakpak API or OTLP export):

| Attribute | Example | Source |
|-----------|---------|--------|
| `code.file.path` | `stakai-0.3.38/src/client/mod.rs` | tracing crate |
| `code.module.name` | `stakai::client` | tracing crate |
| `gen_ai.operation.name` | `"chat"` | StakAI |
| `gen_ai.provider.name` | `"anthropic"` | StakAI |
| `gen_ai.request.model` | `"claude-opus-4-6"` | StakAI |
| `gen_ai.request.temperature` | `"0"` | StakAI |
| `user.name` | `"george"` | telemetry_metadata |
| `user.id` | `"48181a62-9a1a-11ef-9f94-db26f9e6824b"` | telemetry_metadata |
| `session.id` | `"7dc626d6-0887-11f1-9046-4b90f3b418e4"` | telemetry_metadata |
| `gen_ai.tool.definitions` | Array of tool schemas | StakAI |
| `gen_ai.usage.input_tokens` | `"36602"` | On finish |
| `gen_ai.usage.output_tokens` | `"629"` | On finish |
| `gen_ai.usage.cache_read_input_tokens` | `"34525"` | On finish (Anthropic) |
| `gen_ai.usage.cache_write_input_tokens` | `"2076"` | On finish (Anthropic) |
| `gen_ai.response.finish_reasons` | `["Stop"]` | On finish |

**Note:** When using Stakpak API, the server may inject `user.id`, `user.name`, `session.id` into spans. For local/BYOK, Task 1 ensures the CLI populates these.

### 5.2 Tool Call / Tool Response Structure

**Input — tool_call_response (result from a previous tool):**

```json
{
  "id": "toolu_01CpTivovJVvUbD4UvAXskdK",
  "result": "🤖 Dynamic Subagent Created\n\nTask ID: 8bhg86\nDescription: Discover Azure cloud config\n...",
  "type": "tool_call_response"
}
```

**Output — text + multiple tool_calls:**

```json
{
  "parts": [
    { "type": "text", "content": "13 of 14 done, one still running..." },
    {
      "type": "tool_call",
      "id": "toolu_018EAtLhjxBLB8rozTe33JW7",
      "name": "stakpak__get_task_details",
      "arguments": { "task_id": "8bhg86" }
    },
    {
      "type": "tool_call",
      "id": "toolu_01UgUJqGESYGfYrgdqbouuZ7",
      "name": "stakpak__get_task_details",
      "arguments": { "task_id": "nxwgsk" }
    }
  ],
  "finish_reason": "Stop"
}
```

### 5.3 Sub-Agent Tools and Correlation

| Tool | Purpose | Key IDs | Observability Needs |
|------|---------|---------|---------------------|
| `stakpak__dynamic_subagent_task` | Creates subagent with description, instructions, tools, sandbox | Returns `task_id` (e.g. `8bhg86`) in result | Correlate subagent run to parent; record `task_id`, `description`, `enable_sandbox` on span |
| `stakpak__get_task_details` | Gets status/output of background/subagent task | `task_id` in arguments | Link `task_id` to parent subagent creation |
| `stakpak__get_all_tasks` | List all background tasks | — | Useful when agent polls after timeout |
| `stakpak__wait_for_tasks` | Wait for tasks to complete | `task_ids` in arguments | Errors like `[AWAIT_TASKS_ERROR] Task timeout` should be captured |

**Sub-agent flow:** Master agent calls `dynamic_subagent_task` → receives `task_id` → later calls `get_task_details(task_id)` to poll. For trace correlation, record `task_id` (and optionally `description`) on tool execution spans (Task 4).

---

## 6. Tool Calls: Clarification

| Question | Answer |
|----------|--------|
| Are tool calls included in traces? | **Yes.** They are in `gen_ai.input.messages` (tool_call, tool_call_response) and `gen_ai.output.messages` (tool_call parts). |
| Are there separate spans per tool? | **No.** Only attributes on the GenAI span. Task 4 proposes adding them. |
| Does Langfuse see tool calls? | **Yes**, when Langfuse ingests OTLP — the GenAI span’s attributes include tool call data. |

---

## 7. Implementation Considerations

### 7.1 Task 4 Extension: Sub-Agent-Aware Tool Spans

When adding tool execution spans (Task 4), include sub-agent correlation:

```rust
let span = tracing::info_span!(
    "tool.execute",
    "gen_ai.tool.name" = %tool_name,
    "gen_ai.tool.call_id" = %tool_call_id,
    "stakpak.task_id" = tracing::field::Empty,
);
if tool_name == "stakpak__get_task_details" {
    if let Some(task_id) = extract_task_id_from_args(&tool_args) {
        span.record("stakpak.task_id", task_id.as_str());
    }
}
```

For `dynamic_subagent_task` **results**, the `task_id` is in the result string (e.g. "Task ID: 8bhg86"). Optionally parse and propagate as baggage or a follow-up span attribute for downstream correlation.

### 7.2 Error Handling in Tool Results

Tool results can contain errors, e.g. `[AWAIT_TASKS_ERROR] Failed to await tasks: Task timeout`. These appear in `gen_ai.input.messages` as `tool_call_response.result`. Consider:

- Recording tool result status (success/error) on tool execution spans when Task 4 is implemented.
- Ensuring error strings are not stripped from span attributes (may need length limits for very large outputs).

### 7.3 gen_ai.tool.definitions Size

`gen_ai.tool.definitions` is a large JSON array (20+ tools with full schemas). This is useful for Langfuse but can inflate span size. Current behavior records it in full; consider making it opt-in via config if storage/bandwidth is a concern.

---

## 8. File Reference

### 8.1 Files to Modify

| File | Tasks |
|------|-------|
| `libs/shared/src/models/llm.rs` | 1 – Add `telemetry_metadata` to LLMInput, LLMStreamInput |
| `libs/shared/src/models/stakai_adapter.rs` | 1 – Pass `telemetry_metadata` into GenerateRequest |
| `libs/api/src/client/provider.rs` | 1 – Build and pass telemetry_metadata; 3 – Verify headers |
| `cli/src/commands/agent/run/mode_interactive.rs` | 2 – Populate metadata |
| `cli/src/commands/agent/run/mode_async.rs` | 2 – Populate metadata |
| `cli/src/commands/acp/server.rs` | 2 – Populate metadata |
| `cli/src/commands/agent/run/tooling.rs` | 4 (optional) – Tool execution spans |

### 8.2 Out of Scope (Agent Repo)

- **Langfuse SDK** — Not used here; OTLP is the transport.
- **Cost tracking** — No cost logic; server/API handles billing.
- **Session creation** — Handled by API or local storage; this repo consumes session IDs.

---

## 9. Implementation Order

1. **Task 1 (Metadata plumbing)** — Add `telemetry_metadata` to types and adapter; build from `ctx` in provider.
2. **Task 2 (Populate metadata)** — Add user/capability metadata at mode_interactive, mode_async, ACP.
3. **Task 3 (Headers)** — Verify session header propagation.
4. **Task 4 (Tool spans)** — Optional; add when needed for finer observability.

---

## 10. References

- [OpenTelemetry GenAI Semantic Conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/)
- [StakAI tracing](libs/ai/src/tracing.rs) — `record_telemetry_metadata`, `record_input_messages`, `record_streamed_response`
- [StakAI examples](libs/ai/examples/tracing_otel.rs) — OTLP setup with Jaeger
- `AGENTS.md` — Workspace structure and message flow
