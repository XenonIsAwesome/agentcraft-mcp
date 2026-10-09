# Step 4: In-game appearance customization

Status: planned. Depends on [step 3](03-custom-adapters.md).
[Overview](README.md) · [Next](05-executable.md)

## Outcome

Make AgentCraft's appearance configurable independently of the connected harness. The
current warm/orange Claude styling becomes a selectable preset instead of a fixed identity
for every session. Users configure the theme in Minecraft through an `/agentcraft`
subcommand, with live preview and persistent settings. This is a mod presentation feature;
it works with Foreman offline and with simulator, managed, and session execution modes.

## In-game commands and settings

Proposed command surface:

```mcfunction
/agentcraft customize
/agentcraft customize theme neutral
/agentcraft customize theme claude
/agentcraft customize accent "#4F7CAC"
/agentcraft customize reset
```

The bare subcommand opens an appearance screen. Include preset selection, an accent color
picker with a hex field, background/text previews, and Apply, Cancel, and Reset actions.
Provide tab completion and readable command feedback. The command forms update the same
configuration service as the screen. Invalid preset IDs/colors leave the active theme intact.

Ship a provider-neutral preset and a Claude/warm preset that preserves the current look.
New installations default to neutral; existing installations retain their warm appearance
through a one-time migration. User choices are explicit: attaching Claude, Codex, OpenCode,
or Antigravity must not silently change the selected theme. Do not require a preset for
every provider before completing this step.

The screen previews representative buttons, a nameplate, monitor text, task cards, and
decision controls. Apply saves the validated theme; Cancel restores the previous snapshot.
Reset returns to the documented default and is available even with a malformed config.
Persist only applied changes, not every mouse movement in the color picker.

## Current implementation and integration points

- `mod/src/main/java/dev/agentcraft/command/AgentCraftCommands.java` supplies an extensible
  `/agentcraft` server command root, currently gated by game-master permission.
- `mod/src/client/java/dev/agentcraft/client/ui/UiStyle.java` loads bundled style/palette
  JSON into static maps and exposes hardcoded constants such as `CLAY` and `CLAY_DARK`.
- `Kit.java`, `Panels.java`, and `WorldUi.java` share GUI-atlas sprites between screens and
  in-world displays. Primary buttons and other assets have baked colors.
- `mod/src/client/java/dev/agentcraft/client/agents/Nameplate.java` hardcodes provider chip
  colors and caches plate data; changing only the palette JSON will not update every surface.
- `assets-src/palette.json`, `assets-src/ui-style.json`, `assets-src/gen/gui.py`, and
  `assets-src/sync.py` define/generate/sync the shipping palette and sprite assets.

Inspect remaining renderers, HUD/console screens, generated block textures, and static
color caches during implementation.

## Theme model and application

Introduce a client theme service with versioned configuration and an immutable resolved
palette. Suggested semantic roles include accent, accent-hover/pressed, on-accent, surface,
surface-inset, text, muted-text, border, and focus. Keep task status, diff add/delete, and
agent identity as separate semantic families rather than replacing every orange hex value
with the chosen accent. Theme-controlled provider badges must not default every unknown
harness to Claude orange; badge text remains the actual reported harness/model.

Resolve the selected preset plus user overrides into one palette. Validate the complete
palette, then publish it on the client thread with a theme revision. Replace direct color
constants and long-lived cached palette values with semantic lookups or revision-aware
caches. Rebuild affected nameplate/layout/render data on theme changes without recreating
agents, restarting Foreman, or changing tasks.

Resource reload must resolve the same selected theme again and invalidate atlas/render
caches as needed. Use Minecraft's resource manager for theme assets; the current static
classpath load is insufficient for resource-pack overrides and runtime reload behavior.
Keep layout metrics separate from color customization in this milestone.

## Baked sprites: grayscale assets with runtime tint

Use the user's requested Minecraft-style approach: make tintable sprites white/grayscale
and multiply their RGB by the resolved theme color at render time. Preserve the alpha
channel. White maps to the chosen color; darker gray retains shading. Do not generate or
ship a separate texture set for each color/preset and do not regenerate PNGs when users
move the color picker.

Generate neutral grayscale source assets rather than simply tinting the existing orange
PNGs: multiplying an already colored texture cannot reliably produce arbitrary accents.
Use grayscale masks and the same runtime tint principle for tintable decorative block
surfaces. GUI sprites and block models use their respective rendering APIs; do not assume
the block tint registration path automatically colors GUI-atlas sprites.

For multicolor assets, split rendering into a neutral base plus grayscale tint layers/masks.
For example, keep a button's neutral border/base separate from its accent face and render
text with the resolved on-accent token. Tint status dots with their status token, accent
decorations with accent, and agent-specific elements with agent tokens. Intentional fixed
details must stay in an explicitly separate base layer. Never tint the whole screen or
atlas indiscriminately, which would recolor text, portraits, error indicators, and diffs.

Update `Kit`, `Panels`, and `WorldUi` to carry tint colors explicitly through the relevant
sprite/vertex draw calls. Preserve nine-slice metadata, padding, outlines, highlights,
alpha blending, and the existing opaque in-world UI pass. Restore tint state after each
draw where the API uses mutable state, so one widget cannot tint its neighbors. Derive
hover/pressed/disabled colors from the theme instead of retaining orange state variants.

Audit the HQ for branded orange decorative textures. Convert AgentCraft-owned decorative
accents to grayscale masks and runtime block/model tinting where needed, with resource and
chunk-render invalidation on theme revision. Existing worlds should update visually without
rebuilding or replacing blocks. Natural wood, terracotta materials, vanilla blocks, and
character skin art are not blanket color-filter targets. If a specific decorative accent
needs model/texture separation, include that asset conversion in this step rather than
silently leaving it provider-colored.

## Configuration, command routing, and ownership

Store versioned appearance settings in the Minecraft instance's mod config directory,
separately from Foreman's provider/session configuration. Default to client-local settings:
changing your appearance does not rewrite the world, another player's preferences, or
Foreman's profile. Record a legacy migration marker so resets/new installs are deterministic.

Register the local customization subcommand using the appropriate Fabric client command
path while preserving the existing server `/agentcraft` subcommands and their permissions.
Test root-name coexistence, server fallback, suggestions, and permission behavior; do not
remove the existing root's operator requirement merely to let a user change their own UI.
The in-game command is distinct from the Claude/OpenCode `/agentcraft` skill/command and
from step 5's `agentcraft` shell executable.

Theme changes affect presentation only. They do not select a provider, alter agent roles,
grant model permissions, submit goals, or approve merges. No MCP connection or inference
call is needed to preview, apply, reset, or reload a theme.

## Readability and validation

Validate hex syntax and compute suitable on-accent text/shading variants. Preserve readable
contrast across paper screens and dark/in-world displays. Show an accessible alternative
if the requested accent cannot be used with the current foreground. Status and diff meaning
must also remain legible through labels/icons rather than color alone. Never overwrite
error/success meanings with the branding accent.

Add focused resolver/config/migration tests and a small repeatable visual QA scene. Run
asset generator validation and sync checks, build the mod, and capture neutral, warm,
custom-blue, light, and dark accent cases with screen and in-world views. Unit tests alone
cannot prove atlas tint, nine-slice edges, or full-bright world displays render correctly.

## Acceptance

- `/agentcraft customize` opens the settings screen in game without a Foreman connection;
  preset/accent subcommands provide suggestions and immediate feedback.
- Switching warm to neutral/custom updates buttons, selection/focus, HUD, console, monitors,
  boards, nameplates, decision/review screens, and tintable HQ decorations consistently.
- Grayscale assets retain shading/alpha under runtime tint; multicolor layers, nine-slice
  edges, neighboring widgets, status dots, text, and portraits are not unintentionally tinted.
- Applying a theme requires no PNG generation, environment restart, world rebuild, or
  duplicate per-theme texture packs. Reconnect and resource reload retain the selection.
- Apply persists across game restarts; Cancel rolls back preview; Reset and invalid-config
  recovery work. Legacy users keep the warm preset until they choose otherwise.
- Existing `/agentcraft hq`, anchors, and other server subcommands retain their routing and
  permission checks. Local appearance changes do not require operator status.
- Attaching different harnesses leaves the theme unchanged, and mixed-harness badges remain
  truthful and readable. No task/session/worktree state changes during customization.
