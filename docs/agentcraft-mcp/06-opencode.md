# Step 6: Full OpenCode support

Status: planned. Depends on [step 5](05-executable.md).
[Overview](README.md) · [Next](07-codex.md)

## Outcome

A user connects the shared AgentCraft MCP to their OpenCode session, runs `/agentcraft`,
and receives game work automatically through OpenCode's existing server API. OpenCode
continues to own its login, model, conversation, and visible terminal.

## Supported integration path

OpenCode's TUI communicates with its HTTP server. The documented session API supports
`POST /session/:id/prompt_async`, session status, abort, and an event stream. The TUI can
run on a known local port, allowing AgentCraft to connect to that same server. Starting
another `opencode serve` while a TUI already exists creates another server; it is not
automatic attachment to the visible conversation. [Server documentation](https://opencode.ai/docs/server/)

Initially support a known loopback server endpoint and explicit session selection. Add
discovery only where OpenCode exposes reliable metadata. Do not scan arbitrary ports or
silently choose the most recently active session.

## User flow

1. `agentcraft launch --backend session --repo PATH` starts the studio.
2. `agentcraft connect opencode` prints or explicitly installs the MCP config and command.
3. The user starts/uses OpenCode on a known local endpoint and selects the conversation.
4. `/agentcraft` attaches it and grants the intended AgentCraft role(s).
5. A new game goal wakes the conversation; it claims and performs work through shared MCP.

Register HTTP MCP using OpenCode's `mcp` config with `type: remote`, `url`, and local
connection headers. Install the command as `.opencode/commands/agentcraft.md` or the
user-scoped equivalent. Both wrappers use the shared bootstrap resource rather than
embedding a second copy of the coordination workflow.
[MCP configuration](https://opencode.ai/docs/mcp-servers/) ·
[Commands](https://opencode.ai/docs/commands/)

## Adapter implementation

Implement the step 3 contract in a built-in OpenCode adapter. Bind the Foreman connection
to the selected server, project/directory, and session ID. These routing fields are
validated configuration, not arbitrary URLs supplied by a model tool call.

On wake, send a short event-ID-bearing prompt with `prompt_async`. Resolve delivery as
accepted when the server accepts it, then observe session/events for activity. Acceptance
is not job completion. If submission times out after the request was sent, use uncertain
delivery handling, not an immediate new prompt loop.

Consult session status before delivery. Coalesce work hints while busy and deliver at an
idle boundary unless the installed version's steering/queue behavior has been explicitly
tested. Stream/observe status for adapter health without forwarding every token into
Foreman. MCP operations remain the authoritative studio activity stream.

Use OpenCode's supported permission configuration to restrict AgentCraft work to its MCP
execution path when possible, preserving unrelated user config. Keep provider selection
and model settings in OpenCode; do not install an API key into Foreman.

## Cancellation and ownership

Foreman immediately revokes the job lease and stops its owned command operations. Call
OpenCode's session abort endpoint only when the active turn is known to be the
AgentCraft-delivered turn and the binding enabled that capability. Do not abort unrelated
user work that arrived in the same conversation. Otherwise report "AgentCraft access
revoked; harness interruption unavailable" and wait for safe re-entry.

Treat session deletion, server restart, authentication failure, wrong project, or version
mismatch as disconnected/unsupported. Preserve jobs and require explicit rebinding where
identity cannot be proven. Do not spawn a replacement OpenCode process silently.

## Implementation touchpoints

- Add a built-in adapter and fixtures under the shared adapter package/directory.
- Add the command/config renderer to `agentcraft connect opencode` and diagnostics to doctor.
- Reuse session registries, durable cursors, generic MCP tools, and job completion.
- Keep native subagent support optional. A worker sub-session must register its own binding
  and capacity; it cannot bypass Foreman's dependencies/worktree scheduling.

## Acceptance

- Wake a completed/idle conversation in the same visible OpenCode TUI. Confirm the displayed
  session ID is the bound destination and no second hidden server handles the goal.
- Complete a real plan/edit/test/review/user-merge cycle without AgentCraft provider keys.
- Test authenticated server access, project routing, busy delivery, uncertain response,
  duplicate hints, disconnect/reconnect, and Foreman restart.
- Test stop with both AgentCraft and unrelated user activity present. No unrelated turn is aborted.
- Multiple attached OpenCode sessions can work in separate worktrees concurrently, and a
  mixed Claude/OpenCode team uses the same task and decision tools.
- Publish tested OpenCode versions/platforms; validate desktop separately before claiming
  identical support there.
