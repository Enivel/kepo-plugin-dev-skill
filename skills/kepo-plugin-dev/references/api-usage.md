<!--
File purpose:
- Path: references/api-usage.md
- Responsibility: Explain which Kepo plugin public APIs to use in which runtime scenario.
-->

# Public API Usage

Use this reference before adding or changing `@kepoai/provider/api`, `@kepoai/ui/hooks`, `@kepoai/types`, or `widget.config.ts`.

## Runtime Split

Kepo plugins have three public surfaces:

- `widget.config.ts`: declares the plugin contract.
- `src/provider/`: performs background or action-driven work and writes data.
- `src/viewer/`: renders data and handles explicit user interaction.

Keep responsibilities on the correct side. Do not use viewer code for scheduled refresh, and do not use provider code for visual layout decisions.

## `widget.config.ts`

Use `widget.config.ts` to declare what the host needs to know about the widget.

Use fields this way:

- `name`: stable widget name used by the platform.
- `description`: short user-facing widget description.
- `component`: React viewer component. Use `null` only when intentionally relying on the host's generic component.
- `popupComponent`: custom popup UI when the widget needs a custom popup layer. Leave empty when not needed.
- `capabilities`: declare host capabilities the widget explicitly depends on, such as `{ ai: true }` when AI is part of the widget behavior.
- `sizes`: supported Kepo sizes. Only declare sizes that have a real layout.
- `configs`: host settings fields. Use this for user-configurable inputs, filters, switches, and buttons.
- `events`: provider lifecycle handlers such as scheduled refresh.
- `actions`: provider functions called from viewer user actions.

Do not mutate config arrays with `push` during declaration. Define the config structure directly so the CLI parser can convert it reliably.

### Config Options

Use config options for settings that belong in the host settings panel, not inside the widget body.

- `input`: free-text setting such as username, keyword, URL, account name, or threshold.
- `select`: bounded choice or multiple choice such as feed type, category, region, model, or sort mode. Use `onSearch` when options are searched dynamically.
- `switch`: boolean setting such as include archived items, compact mode, or enable alerts.
- `button`: explicit settings-panel operation such as test connection or refresh option list. Do not use it as the primary login entry when login belongs in the widget state.

Use config events for settings-panel behavior:

- `onLoad`: fill or validate initial option state.
- `onChange`: validate or adjust dependent options.
- `onSearch`: search dynamic select options.
- `onClick`: run an explicit settings-panel button action.

Do not use config events as a replacement for scheduled data refresh.

## Provider APIs

Import provider APIs from `@kepoai/provider/api` inside `src/provider/`.

### `Storage`

Use `Storage` to persist widget data that the viewer should render.

Use it for:

- normalized provider results
- refresh metadata such as `updatedAt`
- durable state derived from provider work
- error or status payloads that viewer needs to display

Do not use `Storage` for host settings. Use `Config` or viewer `useConfig` for configuration.

Avoid writing ambiguous empty payloads. If the source returned a login page, challenge page, captcha, blocked response, or abnormal shell, store an explicit status or error state instead of pretending the list is empty.

### `Config`

Use `Config` in provider code when provider execution needs current host settings or needs to update config metadata.

Use it for:

- reading configured username, keyword, feed type, URL, or switches before refresh
- setting validation errors with `setError`
- updating config structure with `updateStruct` when dynamic options change
- reading config option structure with `getStruct`

Do not duplicate configuration into `Storage` unless there is a specific durable derived value that the viewer must render.

Use the config option type as the generic parameter, not the returned value type:

```typescript
const title = await Config.get<InputOption>("title", "");
const tags = await Config.get<SelectOption>("tags", ["news"]);
const enabled = await Config.get<SwitchOption>("enabled", false);
```

Do not write:

```typescript
await Config.get<string>("title", "");
await Config.get<string[]>("tags", ["news"]);
await Config.get<SelectOption>("tags", { defaultValue: ["news"] });
```

For `select`, the value is `string[]` even when the option is single-select. Read the first element when the business value is singular.

### `$fetch`

Use `$fetch` for session-aware HTTP requests when raw response data is enough.

Prefer it for:

- public JSON APIs
- authenticated JSON APIs
- server-rendered HTML that already contains required fields
- raw document responses with embedded data
- lightweight status checks

Use `$fetch.raw` when status, headers, final URL, or non-2xx handling matters.

Do not use `$fetch` when the required data exists only after JavaScript renders the page. In that case use `getPage()` after verifying the raw document is insufficient.

Do not switch to `getPage()` merely because the target requires login. Login decides whether user action or authenticated session is needed; `useBrowser` or rendered-only evidence decides whether the provider needs browser page execution.

### `getPage`

Use `getPage()` only when browser page behavior is required.

Use it for:

- rendered DOM extraction that cannot be recovered from raw source
- login flows triggered by explicit user action
- pages that need browser APIs, cookies, or script execution beyond HTTP fetch
- verifying browser-visible state such as challenge, consent, or login-required pages

Important browser methods:

- `page.show()`: show the browser only after an explicit user action.
- `page.hide()`: hide a browser surface when a background or follow-up step no longer needs it.
- `page.setDevice('desktop' | 'mobile')`: match the target site's required layout.
- `page.getCookies(url?)`: inspect login cookies when needed.
- `page.isVisible()`: poll whether a user-facing login window is still visible.

Do not show the browser during scheduled refresh. If scheduled refresh needs browser page execution, keep it non-visual and store explicit failure states when login or challenge blocks completion.

For website extraction, follow the confirmed acquisition choice:

- `useBrowser: false`: use `$fetch` or `$fetch.raw` and parse the raw HTML/API response.
- `useBrowser: true`: use `getPage()`, wait for the target DOM, then parse page content.

Do not mix these paths without new evidence.

### Provider Events

Use provider events for lifecycle work:

- `onScheduled`: main periodic refresh. It should fetch, parse, normalize, and store data.
- `onCreated`: one-time initialization when the widget is created.
- `onShow`: lightweight work when the widget becomes visible. Do not use it as the main refresh path unless the product explicitly needs visible-time refresh.
- `onHide`: cleanup or pause behavior.
- `onPopupLayerOpened` / `onPopupLayerClosed`: react to popup lifecycle, commonly for login or external flow completion.

Do not put user-triggered button behavior into `onScheduled`; use actions for explicit viewer actions.

For list-like provider results, collect enough valid records before trimming. Do not stop at the first DOM snapshot, first screen, or first selector result when the requirement means recent N items or a feed. Use bounded scroll, pagination, or API pagination where appropriate, record the stop reason, then trim stored arrays after collection. Build Mode currently uses a maximum stored list size of 50 as the default guardrail unless the plugin contract says otherwise.

### Provider Actions

Use `actions` for user-triggered operations from the viewer.

Good action examples:

- refresh now
- mark item read
- open or prepare login flow
- submit a small command
- fetch detail for a selected item

Do not use actions as hidden scheduled jobs. If the operation is periodic, it belongs in `onScheduled`.

### Provider `AI`

Use provider-side `AI.complete` or `AI.stream` when background logic needs AI output before storing a result.

Use it for:

- summarizing fetched content before rendering
- classifying or extracting structured fields
- preparing stored recommendations or insights

When a widget materially depends on AI, declare the widget capability in `widget.config.ts` with `capabilities: { ai: true }`.

Do not expose internal usage, quota, or request metadata in normal widget UI unless the product specifically requires that information.

## Viewer Hooks

Import viewer hooks from `@kepoai/ui/hooks` inside `src/viewer/`.

### `useStorage`

Use `useStorage` to read data written by provider `Storage`.

Use it for:

- displaying refreshed data
- rendering stored error or status states
- lightweight user-local viewer state when it truly belongs to the widget instance

Do not use `useStorage` as the primary source for host settings. Use `useConfig`.

The setter returned by `useStorage` accepts a complete next value or `null`. It is not React `useState`; do not pass a function updater.

```typescript
const [state, setState] = useStorage<State>("state", initialState);
const nextState = { ...state, count: state.count + 1 };
setState(nextState);

// Wrong:
setState((prev) => ({ ...prev, count: prev.count + 1 }));
```

Read each real Storage key separately. Do not invent an aggregate key to avoid multiple hooks.

### `useConfig`

Use `useConfig` to read host settings in viewer code.

Use it for:

- showing selected account, keyword, or mode
- adjusting labels or state based on settings
- deciding whether to show login-required or setup-required guidance

Do not render settings controls in the widget body just because `useConfig` can read the value. Settings controls belong to `widget.config.ts` and the host settings panel.

### `useAction`

Use `useAction` for explicit user-triggered calls to provider `actions`.

Use it for:

- refresh buttons
- mark-read buttons
- login or reconnect buttons
- detail fetches
- commands that must run in provider context

Do not call actions automatically in render loops or effects without a clear user action or lifecycle reason.

Every visible action trigger should show user feedback: loading or disabled state during execution, click feedback, and success or error feedback when applicable.

### `useSizeType` And `useCn`

Use `useSizeType` to choose layout by current widget size.

Use `useCn` to merge Tailwind classes with size-aware variants.

Use them for:

- changing density by size
- hiding lower-priority metadata in smaller sizes
- switching between compact, list, chart, and detail layouts

Do not declare a size in `widget.config.ts` unless the viewer has a real layout for it.

Only implement branches for sizes declared in `widget.config.ts`. Do not add hidden branches for undeclared sizes.

### `useDarkMode`

Use `useDarkMode` when colors, charts, dividers, or image treatments need theme-specific handling.

Do not hardcode a single theme if the widget will render in both light and dark host contexts.

### `usePopupLayer`

Use `usePopupLayer` for explicit popup interactions.

Use browser popup layers for:

- login flows
- account authorization
- opening a target website inside the app browser when the user requested it

Use custom popup layers for:

- larger detail UI
- confirmation flows
- custom forms that are part of the widget experience

Do not open popup layers from background refresh or without user intent.

### `BrowserPane`

Use `BrowserPane` only when the widget or popup needs an embedded browser surface controlled by the viewer.

Use it for:

- explicit browsing surfaces
- login or account flows where the user needs to see and interact with a page
- detail views that intentionally remain inside the app

Do not use `BrowserPane` for hidden scraping. Provider `getPage()` is the provider-side browser API, and `$fetch` is preferred when raw data is enough.

Avoid native `<select>` in final Viewer UI. Use a self-rendered menu/listbox/popover style control when a Viewer interaction really needs a dropdown. Host settings still belong in `widget.config.ts`.

### Viewer `useAI`

Use viewer-side `useAI` when the user explicitly triggers AI work from the UI or when the AI output is part of the immediate interactive viewer experience.

Use provider-side `AI` instead when AI work should happen during scheduled refresh and be stored for later rendering.

When AI is a core widget capability, declare `capabilities: { ai: true }` in `widget.config.ts`.

## Shared Types

Use `@kepoai/types` for shared widget contract types and sizes:

- `IWidget`
- `IPlugin`
- `Size`
- `sizes`
- `sizeScaleMap`

Respect Kepo size semantics:

- `short`
- `small`
- `wide`
- `large`
- `giant`

Declare only sizes the plugin really supports. Smaller sizes should remove lower-priority content instead of shrinking everything until it breaks.

Viewer list rendering rule:

- Provider owns data truncation or storage limits.
- Viewer should render the stored list and use scrolling when needed.
- Do not add `.slice()` in Viewer for list-like data unless the design explicitly says this widget is a preview.
