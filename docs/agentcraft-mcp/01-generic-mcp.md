# Step 1: Generic local HTTP MCP

Status: planned. Depends on no later step. [Overview](README.md) · [Next](02-claude-channel.md)

## Outcome

Any MCP-capable local harness can connect to Foreman, invoke the AgentCraft workflow,
claim jobs, and perform real work without an AgentCraft provider API key. The workflow
supports manual re-entry and bounded waiting. Automatic wake-up after a model turn ends
is delivered by later integrations, not promised by ordinary MCP connectivity.

## User experience

Extend the existing launchers with a proposed `--backend session` selection. Foreman starts
the game WebSocket and an HTTP MCP service and shows connection instructions. The studio
shows "Waiting for session" rather than an API authentication failure. A user configures
their harness, asks it to start AgentCraft, and chooses a profile if several are running.

Before step 2's command installer exists, provide a short copyable bootstrap instruction.
Also expose an MCP prompt and instructions resource for clients that support them; MCP
prompts do not universally create the exact `/agentcraft` command name.

## Implementation

1. Add the official TypeScript MCP SDK as a direct Foreman dependency. Create a separate
   HTTP MCP transport module and a shared tool-registration module. Avoid importing the
   Claude SDK merely to expose generic tools.
2. Wire service start/stop into `foreman/src/main.ts`. Keep the current WebSocket and
   DevBridge ports. Record MCP URL and a versioned instance ID in `runfile.ts` metadata.
3. Add session-mode configuration and status schemas. Separate connection readiness from
   model authentication. Update `protocol.ts`, generated protocol docs, examples, and the
   mod's protocol/status handling together.
4. Refactor reusable scheduling options out of the Claude-specific config where necessary.
   Add an external-session execution driver at the existing Engine boundary. It publishes
   jobs instead of starting a CLI. Keep current managed engines working.
5. Extract a reusable permission service from TeamBackend's private permission gate.
   Expose repository operations through it and log those operations through Foreman.
6. Adapt `agentTools()` to execute against the connection's validated current job and role.
   Preserve the existing handlers and scheduler hooks; do not reimplement their business logic.

Proposed new modules: `foreman/src/mcp/{server,tools}.ts`,
`foreman/src/sessions/{registry,jobs,events,operations}.ts`, and
`foreman/src/agents/session/engine.ts`. Names may change; responsibilities should not.

## Protocol

| Operation | Contract |
| --- | --- |
| `attach_session` | Register a connection, requested studio roles, harness metadata, and optional wake binding; return granted roles and bootstrap context |
| `get_context` | Return bounded goal/task/memory/decision context and protocol version |
| `wait_for_work` | Wait up to a server-capped duration for available work or relevant events after a cursor; disconnect cancels the wait |
| `claim_job` | Atomically grant a job lease if the session has capacity and is authorized for the role |
| `finish_job` | Idempotently finish that job's turn with result/summary; resume existing CI/review flow |
| `detach_session` | Release the connection and begin safe lease revocation |

Retain existing coordination tools: `send_message`, `ask_user`, `write_memory`,
`read_memory`, `update_task`, `report_status`, `list_tasks`, `create_task`, and
`request_merge`. Bind actor identity on the server. Do not let a model choose an arbitrary
actor ID or answer a user's approval decision through an agent tool.

Add a small repository surface: `read_file`, `search_files`, `apply_patch`, `run_command`,
`get_operation`, `wait_operation`, and `get_diff`. Keep path selection scoped to a
server-resolved repository/worktree; command execution must not accept a caller-supplied
replacement environment, git directory, or unrestricted working directory.

A claim returns `{jobId, leaseId, generation, agentId, role, taskId?, cwd, instructions,
prompt, contextVersion}`. Protected tools require the active job and lease. Connection IDs
and MCP transport session IDs are not substitutes for authorization. Resource access and
private memory reads also enforce the connection's granted identity and scope.

## Lifecycle and recovery

- Persist jobs through queued, leased, running, completed, cancelled, and recovery states.
- Keep jobs queued until an eligible session has execution capacity. Do not consume worker
  slots or start the current 45-minute turn timeout merely while waiting for attachment.
- One connection defaults to one active job. A single connection may drive Marlow and a
  worker sequentially. Foreman decides which eligible job it receives next.
- `finish_job` releases capacity before downstream jobs are scheduled. Refuse completion
  with unfinished write/command operations. `update_task(review)` alone is not completion.
- Persist lease generations and invalidate old ones on restart. Reattachment requires a
  fresh handshake and reconciliation, not reuse of an old lease token.
- Heartbeats come from MCP transport activity and supported connection pings. An idle
  model should not spend tokens sending heartbeat tool calls. Lease loss revokes operations;
  reconnect grace and operation cleanup precede reassignment.
- Preserve user questions on accidental disconnect and Foreman shutdown. Reconnect context
  includes their IDs and any answers. Explicit stop/pause/cancel withdraws abandoned prompts
  according to existing semantics.
- Wake/event cursors are durable; an expired journal cursor returns a snapshot/resync marker.

Expose a separate authenticated adapter feed, proposed `GET /harness/events?cursor=...`,
using SSE with bounded replay. Provide binding registration/acknowledgement endpoints
under the same internal namespace. This feed is an AgentCraft API, not a standard MCP
wake-up method. Adapter credentials can access only their authorized bindings/events;
they cannot execute model tools or submit user approvals.

## Commands, questions, and policy

`run_command` returns an operation ID promptly. `get_operation`/`wait_operation` returns
bounded output, offsets, status, and exit code. Commands use Foreman's git-safety/identity
environment and tracked process management; lease revocation terminates owned processes
before a worktree is reused. File operations validate real paths, traversal, and symlink
escapes and enforce the lead's read-only role.

For external sessions, adapt `ask_user` into a durable decision ticket rather than holding
an MCP call open indefinitely. Permissions may similarly create pending operations. The
model checks or awaits the ticket and does not proceed on an unanswered decision. Preserve
blocking behavior for existing managed tools where appropriate.

Do not claim policy can intercept the user's unrelated native harness tools. The shared
workflow mandates MCP repository actions, and later integrations add supported host-side
restrictions. MCP requests cannot approve pending user decisions.

## Local service protection

Bind to loopback, validate Host/Origin, cap payloads and waits, and require a locally
generated connection credential. Store it separately from public discovery metadata with
owner-only permissions or the platform equivalent. Never log it or put it in wake prompts.
This credential pairs local components; it is not an LLM provider API key. Prefer the MCP
SDK's HTTP safeguards and review the actual accepted Origin set per supported client.

## Acceptance

- A session-mode startup succeeds with provider credentials absent and does not spawn Claude
  or Codex. Existing simulator and managed modes retain their behavior.
- An MCP test client initializes, lists schemas, claims jobs, edits an assigned worktree,
  runs tests, reports review, finishes its turn, and observes the existing CI/review flow.
- A user approves the merge through the game; the agent cannot self-approve it.
- Concurrent claims, duplicate completion, lost responses, invalid paths, wrong roles, and
  stale lease calls are tested. Stop during a command prevents worktree reassignment until
  process cleanup is confirmed.
- Restart and reconnect retain tasks, worktrees, messages, and decision answers.
- An idle wait returns within the configured tool timeout, and an expired cursor resyncs.
- `npm run check` passes; mod schema changes build and render correct session status.
