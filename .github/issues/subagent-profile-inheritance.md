# Subagents Do Not Inherit Parent Agent's Profile Configuration

## Summary

When spawning subagents via `stakpak__dynamic_subagent_task`, the subagent process does not inherit the parent agent's profile configuration (e.g., `--profile team`). This causes subagents to fail with API key validation errors when the parent is using a non-default profile.

## Problem

**Observed behavior:**
```
Task ID: fmku1x
Status: Failed
Duration: 0.75s

Output:
❌ API key validation failed: error sending request for url (http://localhost:4000/v1/account)
Please check your API key and run the below command

stakpak login --api-key <your-api-key>
```

**Expected behavior:**
Subagents should inherit the parent agent's profile configuration, including:
- `--profile <name>` setting
- API keys and endpoints from that profile
- Provider configurations

## Root Cause

The subagent spawning mechanism in `stakpak__dynamic_subagent_task` launches a new `stakpak` process but does not pass the current profile context. The spawned process defaults to the `default` profile, which may not have valid API credentials configured.

**Current spawn command (inferred):**
```bash
/opt/homebrew/bin/stakpak -a -...
```

**Required spawn command:**
```bash
/opt/homebrew/bin/stakpak --profile team -a -...
```

## Impact

- **Broken subagent workflows**: Any user with a non-default profile cannot use subagents
- **Team configurations fail**: Organizations using team profiles for shared API keys are blocked
- **Context parameter workaround doesn't work**: Passing `--profile team` in the `context` parameter has no effect since it's just text passed to the subagent's prompt, not the CLI invocation

## Reproduction Steps

1. Configure a non-default profile:
   ```bash
   stakpak auth login --profile team --api-key <key>
   ```

2. Run stakpak with that profile:
   ```bash
   stakpak --profile team
   ```

3. Trigger a subagent task (e.g., ask for parallel exploration)

4. Observe subagent failure with API key validation error

## Proposed Solution

See RFC: `docs/rfcs/rfc_subagent_profile_inheritance.md`

**Key changes:**
1. Pass current profile name to subagent spawn command
2. Propagate relevant environment variables
3. Consider profile inheritance in subagent configuration

## Affected Components

| Component | File | Impact |
|-----------|------|--------|
| Task manager | `libs/shared/src/task_manager.rs` | Subagent spawn logic |
| MCP server tools | `libs/mcp/server/` | `dynamic_subagent_task` implementation |
| CLI main | `cli/src/main.rs` | Profile context availability |

## Workaround

Currently, users must ensure their `default` profile has valid API credentials, or manually set environment variables before running stakpak.

## References

- Related: `rfc-sub-agent-logs-telemetry.md` — Subagent observability improvements
- Related: `server-client-flow.md` — Session and context propagation
