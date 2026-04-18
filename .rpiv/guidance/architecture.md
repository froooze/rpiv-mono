# rpiv-btw

## Monorepo Context
Sibling Pi extension in `rpiv-mono` (npm workspaces). Lockstep version with all `@juicesharp/rpiv-*` packages — never bump independently; use `node scripts/release.mjs <bump|x.y.z>` from repo root. Listed in `extensions/rpiv-core/siblings.ts` so `/rpiv-setup` installs it. Peer-pinned in `packages/rpiv-pi/package.json` as `"*"`.

## Responsibility
Slash-command-only Pi extension implementing `/btw <question>`. Spawns a one-off side call to the same primary model with a read-only clone of the current conversation as context, renders the answer in a bottom-anchored ephemeral overlay, and never writes back to the main agent's transcript or to disk. Per-session `/btw` history is process-scoped via a `globalThis` Symbol-keyed singleton.

## Dependencies
- **`@mariozechner/pi-coding-agent`** (peer): `ExtensionAPI`, `ExtensionContext`, `convertToLlm`, `serializeConversation`, `DynamicBorder`
- **`@mariozechner/pi-ai`** (peer): `completeSimple` (with `tools: []`), `Message`, `UserMessage`, `AssistantMessage`
- **`@mariozechner/pi-tui`** (peer): `Container`, `Spacer`, `Text`, `getKeybindings`, ANSI helpers

## Consumers
- **Pi extension host**: loads via `pi.extensions: ["./index.ts"]`
- **`rpiv-pi`**: lists in `peerDependencies` and `siblings.ts`

## Module Structure
```
index.ts        — Composer: registerBtwCommand + registerMessageEndSnapshot + registerInvalidationHooks
btw.ts          — globalThis-keyed state, system-prompt loader, executeBtw, three registrars
btw-ui.ts       — BtwOverlayController (Component) + showBtwOverlay factory
prompts/
  btw-system.txt — Side-model system prompt (no tools, terse, file:line citations)
```

## No-Pollution Architecture
Five layered mechanisms ensure the main transcript is never touched:
```typescript
// 1) Read-only branch clone via message_end snapshot (or live fallback)
function readBranchMessages(ctx) {
    const cached = getSnapshot(ctx);
    if (cached) return cached.messages;
    const branch = ctx.sessionManager.getBranch();
    return convertToLlm(branch.filter(e => e.type === "message").map(e => e.message));
}

// 2) Direct LLM call bypasses the agent loop — tools: [] is mandatory
const response = await completeSimple(model,
    { systemPrompt: buildSystemPrompt(), messages: buildBtwMessages(ctx, userMessage), tools: [] },
    { apiKey, headers, signal: controller.signal });   // 3) own AbortController, NOT ctx.signal

// 4) Bottom-slot overlay rendered via ctx.ui.custom — no agent-message emission
// 5) Process-scoped, session-keyed storage via globalThis[Symbol.for("rpiv-btw")]
```

## Stable-Reference Prompt-Cache Discipline
```typescript
// History stores actual UserMessage/AssistantMessage object references — never
// re-fabricated. Concatenated as: [branchMessages, ...historyMessages, userMessage]
// so the prefix bytes stay byte-identical for prompt-cache hits.
function buildBtwMessages(ctx, userMessage) {
    const branchMessages = readBranchMessages(ctx);
    const history = getSessionHistory(ctx);
    return [...branchMessages, ...history.flatMap(h => [h.userMessage, h.assistantMessage]), userMessage];
}
```

## Snapshot Invalidation
```typescript
// session_compact / session_tree drop the branch snapshot so the next /btw
// re-derives. Without this, post-compact /btw would see a stale clone.
pi.on("session_compact", async (_e, ctx) => invalidateSnapshot(ctx));
pi.on("session_tree",    async (_e, ctx) => invalidateSnapshot(ctx));
```

## Architectural Boundaries
- **NO disk persistence** — `globalThis` only; lost on Pi exit by design (Decision 4)
- **NO `ctx.signal` reuse** — own `AbortController` so Esc cancels only `/btw`, not the main session (Decision 8)
- **NO tool registration** — slash command + lifecycle hooks only
- **System prompt is frozen** — dynamic context appended via `getCrossSessionHint()`; never mutate the static prefix (cache parity)

<important if="you are customizing the BTW overlay">
## Customizing the Overlay
Touch points in `btw-ui.ts`:
1. **Geometry**: `BTW_OVERLAY_OPTIONS` (anchor, width, maxHeight) + `BTW_MAX_HEIGHT_RATIO`
2. **Glyphs**: `SIDE_PAD`, `ANSWER_PAD`, `BTW_LITERAL`, `PENDING_GLYPH`, `FOOTER_*` constants at file top
3. **Modes**: extend `Mode = "pending" | "answer" | "error"` union and add a setter mirroring `setAnswer`/`setError` (always call `tui.requestRender()`)
4. **Theme**: only via `theme.fg/bg(...)` — never raw ANSI; use `wrapTextWithAnsi`/`truncateToWidth`/`visibleWidth` from pi-tui
5. **Keys**: `matchesKey(data, Key.xxx)` for named keys; Esc must `controller.abort()` + `done()`
6. **Scroll**: natural layout `[banner, "", ...history, echo, "", ...answer, "", footer]` — top-clipped when `natural.length > maxRows` (`scrollOffset=0` shows bottom)
</important>

<important if="you are adding a new side-question variant">
## Adding a Variant
1. Add identity const next to `BTW_COMMAND_NAME` (e.g., `BTW_DEEP_COMMAND_NAME = "btw-deep"`)
2. Add new `MSG_*`/`errXxx` constants in the existing Messages/Errors blocks — never inline strings
3. Drop a new prompt file under `prompts/`; mirror the `readFileSync` + `fileURLToPath(new URL(...))` + `.trimEnd()` recipe
4. Write `registerBtwDeepCommand(pi)` mirroring the existing registrar; wire in `index.ts`
5. Reuse `getState()` storage; pick a different `Map` key or a different `Symbol.for(...)` if isolation required
6. Keep four-branch `StopReason` shape in `executeXxx`: aborted | error | empty | success
7. Always own the `AbortController`; never reuse `ctx.signal`
</important>
