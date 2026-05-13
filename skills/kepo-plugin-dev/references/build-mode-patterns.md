<!--
File purpose:
- Path: references/build-mode-patterns.md
- Responsibility: Summarize reusable Build Mode main-agent and Provider/Viewer subagent patterns for external plugin development workflows.
-->

# Build Mode Agent Patterns

This reference is derived from the Build Mode main agent prompt and its Provider / Viewer subagent prompts in `kepo-server-hono/src/routes/build/prompts/`.

Use it to shape manual plugin work. Do not copy Build Mode's internal Memory/tool names into user-facing messages unless you are editing that system directly.

## Main Workflow Responsibilities

The main workflow owns orchestration and evidence:

- understand the user target
- decide whether the task is a lightweight Viewer-only change or an important contract change
- confirm data source, API contract, login requirement, and extraction rules before implementation
- maintain `widget.config.ts`, root `package.json`, version, release notes, and other project contract files
- split Provider and Viewer work by responsibility
- run build after code changes
- run provider runtime tests after build when provider behavior changed
- inspect logs for runtime bugs
- decide whether to rework contract, Provider, Viewer, or both

Do not delegate build, provider testing, log triage, versioning, release, or contract synchronization to Provider/Viewer implementation work.

## Lightweight Viewer Change

Treat a change as Viewer-only when all of these are true:

- it only changes visual appearance, copy, spacing, typography, density, local layout, or simple animation
- it does not change data source, parser, API contract, login behavior, storage keys, config keys, action names, lifecycle events, size set, or `widget.config.ts`
- current implementation has enough context to identify the target Viewer file

In this case, do not regenerate or redesign the full contract. Patch Viewer only, then run the smallest reliable validation.

If a visual request also changes state design, action behavior, storage shape, config meaning, or multi-size strategy, treat it as an important contract change.

## Provider / Viewer Split

Provider owns:

- data acquisition
- parsing
- cleanup and normalization
- validation
- storage writes
- lifecycle events
- config reads and config structure updates
- provider actions
- auth and login-state handling
- provider-side AI only when the contract enables it

Viewer owns:

- UI structure
- information hierarchy
- visual style
- size adaptation
- user-visible text
- interaction feedback
- popup/browser surfaces explicitly opened by the user
- viewer-side AI only when the contract enables it

When one user request needs both, write different goals for Provider and Viewer. Do not pass the same raw requirement to both.

## Contract Ownership

`widget.config.ts` is the contract source for:

- component imports
- viewer component names
- provider export names
- config keys
- action names
- lifecycle event bindings
- size list
- popup component binding
- capabilities such as AI

Provider and Viewer code must align with this contract.

If implementation reveals that the contract is missing or wrong, stop and update the contract first from the main workflow. Do not let Provider or Viewer silently invent new config keys, action names, event bindings, component names, or sizes.

## Development Gates

Before implementation, confirm the required facts for the route:

- Tool/local-only widget: local behavior, configs, actions, and supported sizes are clear.
- API-backed widget: URL, method, auth, required params, response shape, and parser rules are clear.
- Website-backed widget: target URL, extraction rule, whether browser rendering is required, and login/captcha handling are clear.

Do not begin implementation after a failed data-source analysis or failed search unless the failure has been handled.

For website-backed work, do not guess CSS selectors or login markers from generic words. Use confirmed page evidence or pause for user/browser action.

## Build And Runtime Loop

After Provider or Viewer code changes:

1. Run `pnpm build` from the plugin root.
2. If build fails, classify the error:
   - contract/import/name/package issue: fix contract first
   - Provider-only issue: fix Provider
   - Viewer-only issue: fix Viewer
   - shared type or cross-side issue: fix both deliberately
3. If Provider behavior changed and a widget instance exists, run `kp test-provider-event`.
4. If provider runtime fails, read current logs before another implementation pass.

Do not repeatedly re-run implementation without new evidence from build output, provider test output, logs, or an updated contract.

## Provider Implementation Rules

- Match config keys, event names, action names, provider export names, and storage keys to the contract.
- Use `Config.get<InputOption | SelectOption | SwitchOption | ButtonOption>()` with the config option type, not primitive value type.
- Use `$fetch` for raw HTML/API responses when enough; use `getPage()` only when browser rendering is required.
- Login requirement alone does not imply `getPage()`.
- If login is required, write explicit login-needed/error state rather than ambiguous empty data.
- For list/feed/timeline/ranking data, collect with bounded pagination or scrolling when needed, then trim stored records after collection.
- Add concise diagnostic logs around entry, config, request, parse count, stored count, and failures.

## Viewer Implementation Rules

- Match viewer file path, import name, component name, declared sizes, config keys, action names, and storage keys to the contract.
- Read each storage key directly; do not invent aggregate storage keys.
- `useStorage` setters accept complete values or `null`, not updater functions.
- Render an explicit empty/loading/setup/error state; do not return `null` for empty data.
- Render stored list data with scrolling instead of slicing in the Viewer unless preview-only behavior is explicitly part of the design.
- Implement only declared sizes and make field tradeoffs per size.
- Keep root layout contained: `w-full h-full overflow-hidden flex flex-col relative` is the normal baseline.
- Avoid root-level borders, radius, shadows, full-card opaque backgrounds, and edge-attached glass effects because the host owns the outer container.
- Use `flex-1 min-h-0` for scrollable or remaining-space regions.
- Avoid native `<select>` for final Viewer dropdown interactions.
- Use action buttons with loading/disabled/click feedback.

## AI Boundary

Enable AI only when the product need or contract clearly requires it.

- Provider-side AI: background processing, scheduled processing, or results that must be stored before rendering.
- Viewer-side AI: user-triggered interactive summarize, translate, rewrite, chat, or immediate generation.

When AI is core behavior, ensure `widget.config.ts` declares `capabilities: { ai: true }`.

Do not expose quota, usage, request IDs, or other internal AI metadata in normal UI unless the product explicitly requires it.
