# Step 4: AgentCraft executable and environment launcher

Status: planned. Depends on [step 3](03-custom-adapters.md).
[Overview](README.md) · [Next](05-opencode.md)

## Outcome

Users launch the environment through an `agentcraft` executable instead of knowing which
repository script to run. It starts Minecraft, Foreman, MCP, and enabled Foreman-hosted
adapters, then prints instructions for opening the chosen harness. It does not start model
inference or take over a user session unless the user explicitly selects managed mode.

The initial release can be an npm-installed executable with a `bin` entry and bundled
JavaScript/assets. A standalone native binary is not required by this milestone. Document
Node and Java prerequisites honestly; do not label a development-client launcher a complete
Minecraft distribution.

## Proposed command surface

```sh
agentcraft launch --backend sim
agentcraft launch --backend session --repo /path/to/repo
agentcraft launch --backend session --repo /path/to/repo --profile work
agentcraft launch --backend session --no-game
agentcraft status --profile work
agentcraft stop --profile work
agentcraft connect claude --profile work
agentcraft adapter validate /path/to/adapter.ts
agentcraft adapter enable my-harness --profile work
agentcraft doctor --profile work
```

Keep managed `claude`/`codex` backend options and existing login flags. Session mode should
be the recommended real-work path in documentation; choose a backward-compatible default
for old scripts rather than silently changing their billing/execution behavior. Commands
in this document are proposed, not currently installed commands.

`connect` prints connection instructions by default and supports an explicit install
option for merging harness config and command/skill files. It must not rewrite unrelated
configuration or launch a new inference session. Add later harness targets in steps 5–7.

## Implementation

Extract reusable launch/stop/discovery logic from `tools/unix.mjs`, `tools/launch.ps1`, and
Foreman's `runfile.ts` into cross-platform modules. Keep the existing script entry points
as compatibility wrappers. Preserve Windows process/shim handling and Unix process ownership
checks; do not implement Windows support by shelling out to the Unix launcher.

Separate these operations:

1. Resolve profile, home, repository, game paths, ports, and selected execution mode.
2. Check required runtimes and dependencies. Explain/download only the artifacts necessary
   for the selected launch path; provider credentials are irrelevant in session mode.
3. Start or reuse the matching Foreman. Wait for authenticated health/instance metadata,
   not merely an open TCP port. Refuse incompatible mode/repo/profile reuse.
4. Start Minecraft with the existing Fabric development-client quick-play behavior.
   Preserve support for a separately configured launcher through `--no-game`.
5. Wait for the mod's actual ready/connected state, print MCP URL and next steps, and record
   only processes owned by this invocation.

Use an atomic profile launch lock and instance IDs to prevent two concurrent launches.
On failure, stop newly started owned components in reverse order; never stop a Foreman that
was merely reused. Check process birth identity before sending termination signals.

## Packaging and data

Ship compiled Foreman/CLI code, integration templates, the adapter runner, and versioned
schemas together. Resolve assets relative to the installed package, not the user's current
directory or an assumed source checkout. Put mutable profiles, logs, adapter state,
connection credentials, and launcher metadata under AgentCraft home.

First deliver a source/development-client package with the current Java 25 and Minecraft/
Fabric prerequisites. Resolve the required source/build assets explicitly. A later packaged
mod/standard-launcher distribution can remove the source dependency; it is not implied by
the existence of an executable. Do not distribute Minecraft game binaries as package assets.

Emit human-readable output and a redacted machine-readable summary for integrations. Report
environment ready, harness not connected, adapter blocked, and game disconnected separately.
`doctor` checks profiles, service compatibility, ports, runtimes, harness versions, MCP
initialization, and adapter health without starting an LLM turn.

## Shutdown behavior

`stop` respects ownership and scope (`game`, `foreman`, or both). Stop Foreman-owned adapters
and repository operations and flush state. Do not kill an independently opened Claude,
OpenCode, Codex, or Antigravity session. Notify a harness-hosted bridge that the environment
is offline and allow it to reconnect later. Closing the game alone must not stop Foreman.

## Acceptance

- `agentcraft launch --backend sim` matches the current `node tools/unix.mjs launch
  --backend sim` experience; legacy scripts still work.
- Session launch reaches a ready studio without any provider credential or hidden model
  process. The printed Claude setup works with step 2's integration.
- Validate launch, reuse, failed startup, concurrent launch, stop, and relaunch on Linux,
  macOS, and Windows, including spaces in paths and npm command shims.
- Running outside the checkout resolves installed assets and mutable data correctly.
- `--no-game` supports another Minecraft launcher. CI can exercise `--no-game` without a GUI.
- Stopping one profile does not stop another profile or a user's harness terminal.
