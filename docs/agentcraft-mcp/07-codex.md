# Step 7: Full Codex support

Status: planned. Depends on [step 6](06-opencode.md).
[Overview](README.md) · [Next](08-antigravity.md)

## Outcome

A user-visible Codex session uses the shared AgentCraft HTTP MCP and receives idle work
through a reachable Codex app-server. Codex owns authentication and inference. Preserve
the current managed Codex engine as a separate execution mode.

## Supported attachment arrangement

The public app-server protocol provides `turn/start` for new input on a thread,
`turn/steer` for an active turn, and thread resume/history APIs. It also documents connecting
the terminal UI to an app-server endpoint with `codex --remote`. WebSocket transport is
currently experimental, so compatibility must be pinned and tested.
[App-server documentation](https://developers.openai.com/codex/app-server/)

First support a user-started/reused reachable app-server with the Codex terminal attached
to it. For example, using a currently documented arrangement:

```sh
codex app-server --listen ws://127.0.0.1:4500
codex --remote ws://127.0.0.1:4500
```

The adapter must connect to that same runtime and explicitly selected thread. It must not
start a second private stdio app-server and claim that it has woken the user's terminal.
Use supported local authentication where available, keep endpoints loopback, and keep
secrets out of command-line arguments and discovery output.

An arbitrary existing Codex desktop/CLI session may not expose an attachment endpoint.
Check reachability and ownership; do not manipulate rollout files or undocumented desktop
internals to inject prompts. Add desktop to the full-support matrix only after its public
connection path and live updates are verified. Unsupported arrangements receive clear
shared-server setup instructions.

## MCP and instruction setup

`agentcraft connect codex` renders the MCP config and a local AgentCraft skill. Codex
supports HTTP MCP with connection headers/token configuration and loads repository skills
under `.agents/skills`. Its documented explicit skill entry uses `$agentcraft` or `/skills`;
preserve the native invocation rather than promising an exact `/agentcraft` alias.
[MCP documentation](https://developers.openai.com/codex/mcp/) ·
[Skills documentation](https://developers.openai.com/codex/skills/)

Use the same bootstrap instructions as Claude/OpenCode. The skill attaches a connection
and grants roles, then recovers context and claims jobs. Ensure the selected app-server
thread actually loads the AgentCraft MCP configuration before accepting a binding as ready.
Do not alter the user's model or create a new thread without explicit selection.

## Adapter and RPC work

Refactor `foreman/src/agents/codex/rpc.ts` into a transport-neutral JSON-RPC client plus
owned-process and attachment transports. Keep the current managed engine's private stdio
behavior working. Add initialize/initialized negotiation, protocol/version diagnostics,
notification subscriptions, pending-request cancellation, and disconnect handling for the
reachable transport.

The built-in adapter implements step 3's contract. Binding records include runtime identity,
thread ID, project/cwd, connection generation, and any AgentCraft-owned active turn ID.
Do not use a resume call on a second runtime as a substitute for attaching to the live one.

When the thread is idle, send `turn/start` with a minimal AgentCraft wake prompt. When busy,
prefer queue/coalescing until idle; only use `turn/steer` with the exact expected active
turn ID and explicitly tested/selected behavior. Handle a stale turn ID by refreshing
state, not by spraying new turn requests. Foreman still requires a job lease before any
task mutation or repository operation.

Multiple client subscriptions and approval ownership need a live integration test. The
user's Codex UI must remain the intended handler of native approval requests. The adapter
must not answer those requests automatically or compete with the terminal for them. Core
AgentCraft approvals remain Foreman decisions shown in the game.

## Cancellation, recovery, and telemetry

Revoke the lease and stop Foreman's repository operations first. Interrupt a Codex turn
only when its recorded identity matches the AgentCraft-started turn. Never kill a reused
server or user's terminal. On reconnect, initialize again, verify the same runtime/thread,
reconcile active turn state, and replay undelivered wake hints safely.

Keep available model/token telemetry separate from authorization. Do not estimate dollar
costs from a subscription session or show guessed token counts. Treat native tools outside
the MCP execution path as outside Foreman's interception; configure supported restrictions
where practical and document actual coverage.

## Acceptance

- An idle thread in a visible Codex terminal receives a game goal through the attached
  app-server and calls the generic AgentCraft tools in that same conversation.
- The user sees progress and can interact through their Codex UI; no hidden replacement
  thread/process is created to fake attachment.
- Complete the real worktree/test/review/user-approved merge cycle with Codex's own login.
- Validate two-client notifications and approval ownership, busy delivery/steering,
  cancellation, connection loss, runtime restart, and stale binding/turn rejection.
- Existing managed Codex tests and mixed Claude/Codex team tests still pass.
- Publish the tested Codex versions, transport, operating systems, and terminal/desktop
  surfaces. Experimental transport support and unsupported desktop attachment are explicit.
