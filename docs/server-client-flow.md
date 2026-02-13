# Server-Client Flow: Messages, Tool Calls, and Observability

This document explicitly explains how messages, tool calls, sessions, and observability flow through the Stakpak Agent (CLI) and its backend services.

---

## 1. Architecture Overview

### 1.1 Two Modes: Server (Stakpak API) vs Local

| Aspect | **Server Mode** (Stakpak API) | **Local Mode** (BYOK) |
|--------|-------------------------------|------------------------|
| **LLM inference** | StakAI → StakpakProvider → Stakpak API (apiv2.stakpak.dev) | StakAI → Direct provider (Anthropic, OpenAI, Gemini) |
| **Sessions / checkpoints** | Stakpak API (remote DB) | Local SQLite (~/.stakpak/data/local.db) |
| **Model resolution** | `find_model(..., use_stakpak: true)` → `stakpak/claude-sonnet-4-5` | `find_model(..., false)` → `anthropic/claude-sonnet-4-5` |
| **Tracing** | OTLP spans (StakAI) → exported to configured collector (e.g. Langfuse OTLP) | Same — StakAI spans regardless of provider |

**Important:** LLM inference **always** goes through StakAI. When using Stakpak API, the model is routed to the `stakpak` provider, which forwards requests to the Stakpak API. When local, StakAI talks directly to Anthropic/OpenAI/Gemini.

### 1.2 Component Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              STAKPAK CLI                                         │
├─────────────────────────────────────────────────────────────────────────────────┤
│  main.rs                                                                         │
│       │                                                                          │
│       ▼                                                                          │
│  run_interactive() / run_async() / ACP server                                    │
│       │                                                                          │
│       ▼                                                                          │
│  AgentClient (libs/api)                                                          │
│       │                                                                          │
│       ├── chat_completion_stream() / chat_completion()                           │
│       │       │                                                                  │
│       │       ├── Hooks: BeforeRequest → BeforeInference → AfterInference → AfterRequest
│       │       │   (TaskBoardContextHook builds LLMInput from messages)           │
│       │       │                                                                  │
│       │       └── run_agent_completion()                                         │
│       │               │                                                          │
│       │               ▼                                                          │
│       │         StakAIClient (stakai_adapter)                                    │
│       │               │                                                          │
│       │               ├── chat_stream(LLMStreamInput) / chat(LLMInput)           │
│       │               │                                                          │
│       │               ▼                                                          │
│       │         stakai Inference.generate() / stream()                           │
│       │               │                                                          │
│       │               ├── StakpakProvider  ──► Stakpak API (server mode)         │
│       │               ├── AnthropicProvider ──► Anthropic API (local)            │
│       │               ├── OpenAIProvider   ──► OpenAI API (local)                │
│       │               └── GeminiProvider   ──► Google API (local)                │
│       │                                                                          │
│       └── Session storage: StakpakStorage (API) or LocalStorage (SQLite)         │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Message Flow

### 2.1 End-to-End Message Pipeline

Messages pass through several transformation layers before reaching the LLM API:

```
┌──────────────────┐     ┌───────────────────────┐     ┌────────────────────┐
│  User / TUI      │     │  mode_interactive /   │     │  AgentClient       │
│  UserMessage     │────►│  run_async / ACP      │────►│  chat_completion_  │
│  SendToolResult  │     │  messages: Vec<       │     │  stream()          │
│                  │     │  ChatMessage>         │     │                    │
└──────────────────┘     └───────────┬───────────┘     └─────────┬──────────┘
                                     │                           │
                                     │ sanitize_tool_results()   │
                                     │ (dedup, remove orphans)   │
                                     ▼                           │
                            ┌────────────────────┐               │
                            │  HookContext       │               │
                            │  state.messages    │               │
                            └─────────┬──────────┘               │
                                      │                           │
                                      │ LifecycleEvent::          │
                                      │ BeforeInference           │
                                      ▼                           │
                            ┌────────────────────┐               │
                            │ TaskBoardContext   │               │
                            │ Hook               │               │
                            │ - reduce_context_  │               │
                            │   with_budget()    │               │
                            │ - merge, dedup     │               │
                            └─────────┬──────────┘               │
                                      │                           │
                                      │ ctx.state.llm_input =     │
                                      │   LLMInput { messages:    │
                                      │   Vec<LLMMessage>, ... }  │
                                      ▼                           │
                            ┌────────────────────┐               │
                            │ run_agent_         │◄──────────────┘
                            │ completion()       │
                            └─────────┬──────────┘
                                      │
                                      │ LLMStreamInput {
                                      │   model, messages,
                                      │   tools, headers, ...
                                      │ }
                                      ▼
                            ┌────────────────────┐
                            │ StakAIClient       │
                            │ stakai_adapter     │
                            │ to_stakai_message()│
                            └─────────┬──────────┘
                                      │
                                      │ GenerateRequest {
                                      │   messages: Vec<Message>,
                                      │   options, tools, ...
                                      │ }
                                      ▼
                            ┌────────────────────┐
                            │ stakai Inference   │
                            │ generate/stream    │
                            └─────────┬──────────┘
                                      │
                                      │ Provider-specific conversion
                                      │ (e.g. AnthropicMessage)
                                      ▼
                            ┌────────────────────┐
                            │ LLM Provider API   │
                            │ (Anthropic, etc.)  │
                            └────────────────────┘
```

### 2.2 Key Types

| Type | Location | Purpose |
|------|----------|---------|
| `ChatMessage` | `libs/shared/models/integrations/openai.rs` | OpenAI-shaped: role, content, tool_calls, tool_call_id |
| `LLMMessage` | `libs/shared/models/llm.rs` | Provider-neutral: role, content (String or List of TypedContent) |
| `LLMMessageTypedContent` | `libs/shared/models/llm.rs` | Text, ToolCall, ToolResult, Image, Document |
| `Message` (StakAI) | `libs/ai` | StakAI internal format |
| `AnthropicMessage` | `libs/ai/providers/anthropic` | Anthropic API format |

### 2.3 Defense-in-Depth for Tool Results

Invalid message sequences (orphan tool_results, duplicates) cause Anthropic 400 errors. The codebase uses three layers:

1. **Source prevention** (`mode_interactive.rs`): Avoid pushing cancelled tool_results when retry will send the real one.
2. **Pre-API sanitization** (`sanitize_tool_results`): Dedup and remove orphans from `Vec<ChatMessage>` before every API call.
3. **Context manager** (`task_board_context_manager.rs`): Merge consecutive same-role messages and dedup tool_results in `reduce_context()`.

---

## 3. Tool Call Flow

### 3.1 Interactive Mode: Tool Execution Loop

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  AI returns tool_calls [A, B, C] in stream                                       │
└─────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  stream.rs: ToolCallAccumulator processes deltas                                 │
│  → final tool_calls attached to ChatMessage                                      │
└─────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  mode_interactive: tools_queue = [A, B, C]                                       │
│  Pop A, send AcceptTool(A) to TUI                                                │
└─────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  TUI: User AcceptTool(A) or RejectTool(A)                                        │
└─────────────────────────────────────────────────────────────────────────────────┘
                                      │
                          (if accepted)
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  tooling.rs: run_tool_call(A, session_id, ...)                                   │
│  - MCP tools: call MCP server                                                    │
│  - Local tools: run_command, view, etc.                                          │
└─────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  Push tool_result(A) to messages                                                 │
│  Pop B from queue, send to TUI                                                   │
│  ... repeat until queue empty ...                                                │
└─────────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│  All tools done → has_pending_tool_calls = false                                 │
│  → Fall through to next API call with full message history                       │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Tool Call Accumulation (Streaming)

In `cli/src/commands/agent/run/stream.rs`:

- `ToolCallAccumulator` receives `ToolCallDelta` events.
- Supports **ID-based** matching (Anthropic/StakAI) and **index-based** fallback (OpenAI).
- Accumulates `name` and `arguments` across deltas.
- On stream end: `into_tool_calls()` returns complete `Vec<ToolCall>`.

### 3.3 Cancel/Retry Flow

When a tool is cancelled (retry/shell mode):

- If queue is **non-empty**: push `TOOL_CALL_CANCELLED` placeholder to keep message chain valid.
- If queue is **empty**: do not push; the shell/retry flow will send `SendToolResult` later with the real result.

### 3.4 Sub-Agent Tools

| Tool | Purpose | Key IDs |
|------|---------|---------|
| `stakpak__dynamic_subagent_task` | Creates subagent with description, instructions, tools, sandbox | Returns `task_id` (e.g. `8bhg86`) in result |
| `stakpak__get_task_details` | Gets status/output of background/subagent task | `task_id` in arguments |
| `stakpak__get_all_tasks` | List all background tasks | — |
| `stakpak__wait_for_tasks` | Wait for tasks to complete | `task_ids` in arguments |

**Flow:** Master agent calls `dynamic_subagent_task` → receives `task_id` → later calls `get_task_details(task_id)` to poll. Tool results can include errors, e.g. `[AWAIT_TASKS_ERROR] Failed to await tasks: Task timeout`.

---

## 4. Observability and Tracing

### 4.1 Stack: No Langfuse SDK in This Repo

This agent repo does **not** use the Langfuse SDK. Tracing is done via:

- **tracing** crate
- **tracing-opentelemetry** (when `tracing` feature is enabled on stakai)
- **OpenTelemetry (OTLP)** — spans can be exported to any OTLP-compatible backend

Langfuse can ingest traces by configuring its **OTLP endpoint**. Traces from this CLI will appear in Langfuse when:

1. The app configures an OTLP exporter (e.g. Jaeger, or Langfuse OTLP).
2. StakAI is built with the `tracing` feature.

### 4.2 Where Spans Are Created

| Location | Span / Behavior |
|----------|------------------|
| `libs/ai/src/client/mod.rs` | `generate()` and `stream()` create `tracing::info_span!` with `gen_ai.*` attributes |
| `libs/ai/src/types/stream.rs` | On `StreamEvent::Finish`, records usage and tool calls on the span |
| `libs/ai/src/tracing.rs` | Helpers: `record_input_messages`, `record_streamed_response`, `record_telemetry_metadata` |

### 4.3 GenAI Span Attributes (OpenTelemetry Semantic Conventions)

| Attribute | Description | When Set |
|-----------|-------------|----------|
| `gen_ai.operation.name` | "chat" | Start of generate/stream |
| `gen_ai.provider.name` | e.g. "anthropic", "stakpak" | Start |
| `gen_ai.request.model` | Model ID | Start |
| `gen_ai.request.temperature` | Temperature | Start (if set) |
| `gen_ai.request.max_tokens` | Max tokens | Start (if set) |
| `gen_ai.input.messages` | JSON array of input messages | After request |
| `gen_ai.output.messages` | JSON array of output (incl. tool calls) | On finish |
| `gen_ai.tool.definitions` | JSON array of tool definitions | Start (if tools) |
| `gen_ai.usage.input_tokens` | Prompt tokens | On finish |
| `gen_ai.usage.output_tokens` | Completion tokens | On finish |
| `gen_ai.usage.cache_read_input_tokens` | Cache hit tokens | On finish (Anthropic) |
| `gen_ai.usage.cache_write_input_tokens` | Cache miss tokens | On finish (Anthropic) |
| `gen_ai.response.finish_reasons` | Finish reason array | On finish |

### 4.4 Tool Calls in Traces

Tool calls are **included as span attributes**, not as separate child spans:

| Attribute | Content |
|-----------|---------|
| `gen_ai.input.messages` | Input messages with `tool_call` and `tool_call_response` parts in JSON |
| `gen_ai.output.messages` | Output with `tool_call` parts (id, name, arguments) in JSON |
| `gen_ai.tool.definitions` | Tool definitions (name, description, parameters) in JSON |

When Langfuse ingests OTLP, these attributes appear on the GenAI span. There are **no** dedicated child spans for each tool execution in this repo (optional enhancement; see `docs/rfc-sub-agent-logs-telemetry.md`).

### 4.5 Custom Metadata (telemetry_metadata)

StakAI supports `GenerateRequest.telemetry_metadata: Option<HashMap<String, String>>`. Each key-value is recorded as a span attribute. Example:

```rust
metadata.insert("session.id".to_string(), session_id.to_string());
metadata.insert("user.id".to_string(), user_id.to_string());
metadata.insert("user.name".to_string(), username.to_string());
metadata.insert("gen_ai.capability.name".to_string(), "agent_interactive".to_string());
```

**Current state:** `stakai_adapter` always passes `telemetry_metadata: None`. The RFC proposes wiring this from `provider.rs` and callers.

---

## 5. Langfuse Integration

### 5.1 How Langfuse Receives Traces

| Path | Description |
|------|-------------|
| **OTLP ingestion** | Configure Langfuse OTLP endpoint. StakAI spans (when OTLP exporter is set) flow to Langfuse. |
| **No Langfuse SDK here** | This repo does not call Langfuse APIs directly. |

### 5.2 OTLP Setup Example

See `libs/ai/examples/tracing_otel.rs`. To send traces to Langfuse:

1. Use Langfuse’s OTLP endpoint (if offered).
2. Or export to Jaeger/OTLP collector that forwards to Langfuse.
3. Ensure `stakai` is built with `features = ["tracing"]`.

### 5.3 Session Correlation

For server-side correlation:

- **Header:** `X-Session-Id` is injected in `provider.rs` when `ctx.session_id` is set.
- **Metadata:** `session.id` (and `user.id`, `user.name`) can be added via `telemetry_metadata` for span attributes (RFC proposed).

---

## 6. Session and Checkpoint Flow

### 6.1 Session Creation

| Mode | Where | How |
|------|-------|-----|
| **Server** | Stakpak API | `create_session()` → returns session with id |
| **Local** | SQLite | `LocalStorage` creates session in DB |

### 6.2 Session Propagation

```
mode_interactive / mode_async
    │
    ├── session_id from CLI (-s, --session) or checkpoint resume
    │
    ▼
AgentClient.chat_completion_stream(session_id, metadata)
    │
    ├── HookContext { session_id, ... }
    │
    ├── initialize_session(ctx) → current_session
    │   - Stakpak: API create or resume
    │   - Local: SQLite create or resume
    │
    ├── ctx.set_session_id(current_session.session_id)
    │
    ├── run_agent_completion()
    │   └── input.headers["X-Session-Id"] = session_id  (for API)
    │
    └── save_checkpoint() → checkpoint_id
        └── response.metadata["session_id"] = session_id
```

### 6.3 Response Metadata

`ChatCompletionResponse.metadata` can include:

- `session_id`
- `checkpoint_id`
- `state_metadata` (trimming, etc.)

---

## 7. Entry Points Summary

| Entry Point | File | Flow |
|-------------|------|------|
| **Interactive TUI** | `cli/src/commands/agent/run/mode_interactive.rs` | UserMessage → API → stream → tool loop → repeat |
| **Async / headless** | `cli/src/commands/agent/run/mode_async.rs` | Same client flow; no TUI; auto-approve or pause on approval |
| **ACP (Zed)** | `cli/src/commands/acp/server.rs` | LSP-like protocol; uses same `chat_completion_stream` path |
| **Watch** | `cli/src/commands/watch/` | Scheduled runs; uses async flow |

---

## 8. File Reference

| Concern | Key Files |
|---------|-----------|
| Message types | `libs/shared/models/integrations/openai.rs`, `libs/shared/models/llm.rs` |
| Message conversion | `libs/shared/models/stakai_adapter.rs` |
| Context / trimming | `libs/api/local/context_managers/task_board_context_manager.rs` |
| Hooks | `libs/api/local/hooks/task_board_context/mod.rs` |
| Agent client | `libs/api/client/provider.rs`, `libs/api/client/mod.rs` |
| Tool execution | `cli/commands/agent/run/tooling.rs` |
| Stream processing | `cli/commands/agent/run/stream.rs` |
| Tracing | `libs/ai/src/tracing.rs`, `libs/ai/src/client/mod.rs` |
| OTLP example | `libs/ai/examples/tracing_otel.rs` |

---

## 9. See Also

- [RFC: Sub-Agent Logs and Telemetry](rfc-sub-agent-logs-telemetry.md) — Telemetry metadata, tool span attributes, sub-agent correlation, implementation tasks
