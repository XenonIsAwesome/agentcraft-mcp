# Step 7: Full Google Antigravity support

Status: planned. Depends on [step 6](06-codex.md).
[Overview](README.md)

## Outcome

A user connects Google Antigravity to the shared AgentCraft HTTP MCP, explicitly invokes
the AgentCraft workflow in a selected conversation, and receives game work automatically.
Antigravity owns login, inference, and conversation state. Use its documented sidecar path
for event-driven wake-up.

## Supported surface and mechanism

Antigravity documents HTTP MCP through `mcpServers` entries with `serverUrl` and headers.
Antigravity 2.0 documents lifecycle-managed sidecars and an `agentapi` executable exposed
inside their environment. A sidecar can call:

```sh
agentapi send-message <conversation_id> <prompt>
```

That sends input to an existing conversation. Sidecars are explicitly enabled by the user;
their discovery/configuration and runtime paths are documented by Google.
[MCP documentation](https://antigravity.google/docs/mcp) ·
[Sidecar documentation](https://www.antigravity.google/docs/sidecars/)

Do not assume sidecars are available in every older Antigravity IDE or CLI release. Initial
full automatic support targets the documented Antigravity 2.0 surface. Other surfaces
require their own verified API or use step 1's manual/waiting fallback.

## User flow

1. Launch AgentCraft session mode with the selected repository/profile.
2. `agentcraft connect antigravity` prints or explicitly installs MCP and workflow assets.
3. Enable the AgentCraft sidecar through Antigravity's supported configuration.
4. Open/select the intended conversation and explicitly invoke the AgentCraft skill/workflow.
5. Attach that conversation to the profile. Only after selection does the sidecar deliver
   future AgentCraft wake hints to it.

Use the installed version's documented skill/workflow invocation. Offer `/agentcraft` only
on a surface where custom commands/workflows actually provide that alias. Keep a portable
explicit AgentCraft skill instruction as the fallback; invocation syntax is not part of
the shared MCP contract.

## Adapter hosting and binding

Implement a harness-hosted adapter using step 3's SDK. Antigravity starts/restarts the
sidecar; Foreman exposes the authenticated adapter event/binding API. The sidecar
subscribes to the selected profile's durable event journal and delivers accepted wake
hints using `agentapi`, with explicit argv and no shell interpolation.

The `agentapi` executable is documented as available on the sidecar's PATH. Do not assume
a generic Foreman subprocess can invoke it. Package the sidecar entry point, SDK/runtime
dependencies, and validated configuration with the integration assets produced by step 4.

Resolve conversation identity through a documented host mechanism. Validate that the
selected conversation/project is accessible to the sidecar. If the installed surface
cannot expose a conversation ID programmatically, provide an explicit user selection/
paste step and validate it before enabling wake delivery. Never infer identity by choosing
the newest conversation or reading undocumented internal databases.

Persist the Foreman binding and delivery cursor separately from the sidecar's provider
state. A sidecar restart revalidates both destination and binding generation. An invalid
conversation ID or disabled sidecar must appear as disconnected, not as a lost task.

## Event and execution behavior

Send short hints identifying the event and instructing the model to inspect AgentCraft
through MCP. Coalesce repeated work-available notifications. Where `agentapi` does not
provide a delivery idempotency mechanism or busy/idle status, use the shared uncertain/
at-least-once handling and fenced job claims. Test its actual busy-conversation behavior
before choosing queueing versus immediate delivery.

The model uses the same file, command, coordination, decision, and completion tools as
every other harness. Use supported Antigravity permission configuration where practical;
do not claim Foreman intercepts arbitrary native IDE actions. Native subagents are optional
and must register distinct execution capacity before receiving concurrent jobs.

## Cancellation and ownership

Revoke the AgentCraft lease and stop Foreman-owned operations immediately. The documented
`send-message` command alone does not establish a turn-interruption API. Declare
`interrupt: false` until a supported version-specific interrupt mechanism is verified.
Do not kill Antigravity or the user's conversation to implement cancellation. Report
accurately that AgentCraft access stopped even if the model is still generating text.

Disabling or uninstalling the adapter removes only AgentCraft-owned assets/configuration,
keeps unrelated sidecars and MCP entries, and preserves worktrees, tasks, and conversation
history. Provider credentials are never imported into Foreman.

## Acceptance

- In a tested Antigravity 2.0 installation, an enabled sidecar wakes the selected idle
  conversation after a game goal, and that conversation claims a job through HTTP MCP.
- Complete plan/edit/test/review/user-approved merge without an AgentCraft provider key.
- Test missing `agentapi`, disabled sidecar, inaccessible conversation, busy session,
  duplicate delivery, sidecar crash/restart, Foreman restart, and invalid binding.
- Stop rejects stale MCP mutations and terminates Foreman-owned commands even when the
  harness does not expose interruption. The UI distinguishes those outcomes.
- Mixed Antigravity/Claude/OpenCode/Codex connections share the same board, memory, decisions,
  dependency rules, and worktree isolation, with one lease per executing session.
- Publish tested versions/platforms/surfaces and invocation syntax. Older IDE/CLI variants
  are not included in the full-support claim without live validation.
