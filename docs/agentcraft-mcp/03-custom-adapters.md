# Step 3: User-custom TypeScript wake-up adapters

Status: planned. Depends on [step 2](02-claude-channel.md).
[Overview](README.md) · [Next](04-executable.md)

## Outcome

Users can write and enable a TypeScript file that delivers AgentCraft wake hints to another
harness. Built-in integrations use the same versioned adapter contract. Adapters do not
need to understand task scheduling, implement repository tools, or hold model credentials.

## Responsibilities and boundaries

An adapter discovers or validates an explicitly selected destination, binds it to an
AgentCraft connection, delivers a wake hint, reports connection health, and optionally
supports interrupting AgentCraft work. Foreman owns the journal, retries, leases, job
selection, and completion. Adapters cannot claim jobs or approve decisions on the model's
behalf through their adapter control credential.

Some adapters run under Foreman; others must run under the harness. Claude launches its
stdio channel bridge, and Antigravity provides `agentapi` inside a sidecar. Support both
hosting arrangements instead of forcing every adapter into Foreman's process.

## Proposed TypeScript contract

Publish a small SDK with runtime validation and TypeScript declarations. An illustrative
API, to finalize during implementation:

```ts
type Destination = {
  id: string;
  label: string;
  surface: string;
};

type Binding = {
  id: string;
  generation: number;
  destination: Destination;
};

type WakeEvent = {
  id: string;
  cursor: string;
  bindingId: string;
  bindingGeneration: number;
  kind: "work_available" | "message_available" | "decision_settled" | "cancelled";
  jobId?: string;
};

type Delivery =
  | { status: "accepted"; receipt?: string }
  | { status: "busy"; retryAfterMs?: number }
  | { status: "uncertain"; receipt?: string }
  | { status: "unavailable"; reason: string; retryable: boolean };

export default defineAdapter({
  apiVersion: 1,
  id: "my-harness",
  hosting: "foreman", // or "harness"
  capabilities: { wake: true, interrupt: false },
  async start(ctx) { /* validate config; register owned resources */ },
  async discover(ctx): Promise<Destination[]> { /* optional, scoped discovery */ },
  async bind(ctx, destination): Promise<Binding> { /* explicit selection */ },
  async wake(ctx, binding, event): Promise<Delivery> { /* deliver one hint */ },
  async health(ctx, binding) { /* connected, degraded, unsupported, disconnected */ },
  async unbind(ctx, binding) { /* release resources owned by this binding */ },
  async stop(ctx) { /* release adapter resources */ },
});
```

The SDK defines the omitted context and health types. Context supplies an abort signal,
bounded redacting logger, validated configuration, adapter-private durable state, and an
authenticated client to the narrow adapter event/binding API. An optional `interrupt`
method is permitted only when its capability is declared. This sketch is not executable
code and is not a promise about an existing package name.

The adapter uses provider-specific destination details in private validated state rather
than placing arbitrary endpoints or secrets in model-controlled tool arguments. Foreman
creates binding IDs/generations and confirms which selected destination is authorized.
Human labels and client-reported identity are informational, not proof of authorization.

## Configuration and loading

Proposed configuration:

```json
{
  "session": {
    "adapters": [
      {
        "id": "my-harness",
        "module": "/absolute/path/to/my-harness.ts",
        "enabled": true,
        "options": { "endpoint": "http://127.0.0.1:4096" }
      }
    ]
  }
}
```

Require explicit installation/enabling. Do not auto-execute adapter files found in a
repository, received through MCP, or linked in a task. Resolve file paths deterministically
and validate the export/API version before starting. Initially use the project's existing
TypeScript runtime rather than assuming every Node 22 release runs TypeScript directly.
Step 4 bundles the runner into the executable distribution.

Run user adapters in supervised subprocesses with structured IPC, per-adapter logs,
startup/shutdown timeouts, crash backoff, and a stop circuit after repeated failures.
This contains crashes; it is not an OS security sandbox. A TypeScript adapter is trusted
local code with the privileges of its process, so explain that at installation.

Avoid inheriting unrelated provider secrets into Foreman-hosted adapters. Use an allowlist
for process environment and explicit secret references, not inline values in shared config.
Harness-hosted adapters may receive credentials/environment from their host; document
that boundary and do not claim AgentCraft can restrict the entire host process.

## Delivery, restarts, and cancellation

- Subscribe using a replay cursor. Acknowledge events only after an accepted delivery or a
  durable uncertain-delivery record. Do not equate transport acceptance with model action.
- Retry unavailable/busy delivery with bounded backoff and jitter. Coalesce pending work
  hints for one binding. Do not flood the conversation with repeated prompts.
- An uncertain HTTP result may have been delivered. Reconcile receipts when possible;
  otherwise use at-least-once delivery and rely on fenced job claims for execution safety.
- Revoked bindings cannot receive new events. After restart, revalidate destination and
  generation before replay. Never choose a different conversation merely because it is newest.
- Stop cancels delivery and revokes AgentCraft job access immediately. If the harness has
  no interrupt API, report that fact; never kill a user terminal or mark its model stopped.
- Adapters cannot suppress Foreman's operation cleanup or extend a revoked write lease.

## Built-in migration and author experience

Move step 2's Claude wake behavior into this contract while keeping its stdio proxy host.
The shared workflow, MCP schemas, and connection setup remain compatible. Future OpenCode,
Codex, and Antigravity adapters import the same SDK.

Ship a minimal logging/mock adapter, a local HTTP example, a schema/contract validator,
and a test helper that replays events and checks delivery results. Document discovery,
binding selection, cancellation limits, secrets, and compatibility versioning. No hot reload
in the first version: stop/restart the adapter explicitly and invalidate its old binding.

## Acceptance

- A user can copy an example `.ts` file, configure it, validate it, and deliver a wake hint.
- Built-in Claude support passes the same contract suite after migration.
- Unknown API versions, malformed exports/results, crashes, hangs, duplicate events,
  unavailable destinations, and uncertain delivery are handled deterministically.
- Two profiles and two conversations cannot cross-deliver events or reuse each other's
  binding credentials. Adapter state and logs do not expose authentication tokens.
- The adapter can be disabled without losing tasks, decisions, or the user's conversation.
