---
name: kepo-plugin-dev
description: Develop, debug, validate, locally install, and publish Kepo plugins in the user's current Kepo plugin workspace. Use when Codex needs to work on plugin projects, create a plugin scaffold, run kp CLI commands, fix provider or viewer behavior, trigger test-provider-event, prepare VERSION.md or preview assets, local-publish to the Kepo client, or submit a plugin version for backend review. For new website-backed widget discovery and design-document workflows, delegate to kepo-widget-builder first.
---

# Kepo Plugin Dev

## Overview

Use this skill for Kepo plugin work after the target plugin is known, or when the user asks for plugin development, debugging, local installation, or release. Act as the workflow driver: identify the stage, run the obvious checks, and guide the user only at decision points that change scope, data source, login behavior, or release state.

Do not make the user know the workflow. When enough local context exists, proceed with the next correct action and report concise results in Chinese.

## Relationship To Other Skills

Use this skill for plugin execution work:

- editing an existing plugin
- creating a scaffold when the plugin goal is already clear
- fixing provider, viewer, config, GUIDE, VERSION, preview, or publishing issues
- running `kp` CLI commands
- verifying behavior through the local Kepo client

Use `kepo-widget-builder` first when the task is a new website-backed widget and the site, data source, widget scope, or visual style still needs real browser analysis. After that workflow fixes the plugin root, documents, and data-source decision, resume here for implementation, debugging, local install, and publishing.

## Build Mode Agent Pattern

When a task is large enough to split, follow the Build Mode agent pattern from the app:

- The main workflow owns fact gathering, target scoping, contract decisions, dependency/version decisions, build, provider runtime tests, logs, and final release.
- Provider work owns only data acquisition, parsing, cleanup, validation, storage writes, lifecycle events, config reads, provider actions, login/auth handling, and provider-side AI when the contract enables it.
- Viewer work owns only interface structure, information display, visual style, layout, size adaptation, user-visible copy, action feedback, popup/browser surfaces, and viewer-side AI when the contract enables it.

Do not copy the same raw user request into Provider and Viewer work. Split it by responsibility. If a change only affects visual text, color, spacing, density, layout, or other display behavior, keep it in Viewer. If a change only affects data source, request, parser, storage keys, config keys, actions, lifecycle events, or login behavior, keep it in Provider.

Provider and Viewer implementation must treat `widget.config.ts` as the contract source. If the required config key, action name, event binding, component import, size list, or capability is missing or inconsistent, stop and fix the contract from the main workflow; do not let implementation code silently invent a new contract.

## Default Behavior

Prefer automatic execution over asking process questions.

Start by classifying the request:

- New plugin with clear scope: create `plugins/<plugin-folder>` only by running `pnpm create kepo-plugin <plugin-folder>`, then inspect scaffold files. Never create a new plugin project by copying a template directory or hand-writing the scaffold.
- Existing plugin change: locate the plugin root, read `package.json`, `widget.config.ts`, relevant `src/provider/` and `src/viewer/` files, then patch narrowly.
- Normal development request: after scoped edits and type/build checks, default to `kp dev` for interactive debugging in the local Kepo client. Treat this as the normal handoff point before install or remote release.
- Runtime bug: identify the exact widget instance, run provider verification, inspect logs and stored data, then patch.
- Local install request: build, then run `kp local-publish` or `kp local-publish --force` when a same-version install is expected.
- Backend review request: validate version, release notes, build, preview assets, login, then run `kp publish` and report `status` and `reviewStatus`.

Ask the user only when:

- the plugin folder cannot be identified safely
- the requested widget or data source is ambiguous
- real login or authenticated browser state is required and unavailable
- publishing will submit a version to the backend
- there are unrelated local changes in files that must be edited
- a destructive operation would be required

When asking, ask one direct question in Chinese and keep moving once answered.

## Source Boundaries

Work inside the user's current project workspace unless the user explicitly provides another path.

Do not assume or reference any fixed absolute local path. Discover the project and plugin roots from the current working directory and nearby repository structure.

Do not infer plugin behavior from a neighboring plugin unless the user explicitly names it as a reference. Use these sources first:

- the target plugin folder
- the scaffold baseline in the current project, when one exists
- `@kepoai/*` package types
- `cli-v2` command source when CLI behavior matters
- existing workflow files inside the target plugin, if present

Do not add empty wrapper helpers, keyword lists, alias maps, or fuzzy matching to guess intent. Prefer real fields, manifests, widget IDs, configs, stored data, runtime logs, and actual command output.

## Command Guidance

Read [references/cli-commands.md](references/cli-commands.md) when deciding which `kp` command to run or when explaining command behavior.
Read [references/api-usage.md](references/api-usage.md) before adding or changing `@kepoai/provider/api`, `@kepoai/ui/hooks`, or `widget.config.ts` usage.
Read [references/build-mode-patterns.md](references/build-mode-patterns.md) when splitting Provider/Viewer work or deciding whether to regenerate an implementation plan.

Default plugin commands:

```bash
pnpm install
pnpm typecheck
pnpm build
kp dev
kp test-provider-event --widget-id <widgetId> --event-name onScheduled
kp local-publish
kp local-publish --force
kp publish
```

Run commands from the plugin root unless the command specifically belongs to the CLI source package or the project workspace root.

Remember:

- `pnpm build` in scaffolded plugins usually runs `pnpm typecheck && kp build`.
- `kp dev` is the default live testing handoff after ordinary development work. It builds into `.kepo/dev`, watches files, notifies the running Kepo client, and tails dev logs. Once it is running, keep it alive for user testing and hot updates unless the user asks to stop it, debugging is finished, or a release/install step requires stopping it. If `kp dev` is already running while you make later edits, prefer reusing the existing watcher instead of restarting it. If `kp dev` fails to connect to the local socket, the Kepo main app may not be running or may have lost its socket; tell the user to restart the Kepo main app and then retry `kp dev`, instead of treating that as a plugin code failure.
- `kp build -p` packages `.kepo/prod` output into `dist/*.kp`.
- `kp publish` is interactive because it asks for a changelog.
- `kp test-provider-event` talks to the running local Kepo client through socket; it is the main provider verification tool.
- `reviewd` is not registered in the public `kp` command entrypoint.

## Workflow

### 1. Intake And Locate

Find the plugin root before changing files.

Check:

- `plugins/<name>/package.json`
- `plugins/<name>/widget.config.ts`
- `plugins/<name>/src/provider/`
- `plugins/<name>/src/viewer/`
- `plugins/<name>/VERSION.md`
- `plugins/<name>/GUIDE.md`
- `plugins/<name>/.kepo-workflow/STATE.md` when it exists
- current-directory equivalents such as `package.json`, `widget.config.ts`, `src/provider/`, and `src/viewer/` when the current directory is already a plugin root

If no plugin root is clear, ask for the plugin folder or target plugin name. Do not guess from similar names.

### 2. Create Or Prepare

For a new plugin with a known folder name:

```bash
cd <discovered-plugin-workspace>
pnpm create kepo-plugin <plugin-folder>
```

New plugin projects must be created only through the scaffold command. Do not use `cp -R` from a template directory, do not manually create baseline scaffold files, and do not treat a copied template as an acceptable new project. If the scaffold command cannot be run because of cwd, permissions, network, or an existing folder, stop and report the blocker before making files.

After scaffold or before edits, verify the baseline:

- `package.json` has `pluginId`, `alias`, `version`, `description`, scripts, and `@kepoai/cli`
- `widget.config.ts` defines real widgets, sizes, configs, events, and actions
- `src/provider/` and `src/viewer/` match the widget contract
- `GUIDE.md` exists when build or publish needs it

### 3. Implement

Keep changes narrow.

Use the public APIs according to their runtime side:

- Provider APIs belong in `src/provider/` for data acquisition, storage writes, configuration validation, browser-backed collection, provider actions, and provider-side AI calls.
- Viewer hooks belong in `src/viewer/` for rendering stored data, reading configs, calling explicit actions, adapting layouts to widget size and theme, and opening explicit popup or browser surfaces.
- `widget.config.ts` is the contract file that connects viewer, provider events, provider actions, supported sizes, config options, popup components, and capability declarations.

Do not move a responsibility to another side just because the API is easier to call there. For example, do not put host settings controls in the viewer, do not run scheduled refresh logic from the viewer, and do not use provider browser APIs for visual-only rendering decisions.

Provider work:

- use actual config fields and widget runtime inputs
- log target URL, parsed count, stored count, stop reason, and explicit rejection reasons when debugging
- for lists, feeds, timelines, rankings, notifications, or grids, do not treat the first visible page as complete; use bounded scroll, pagination, or real APIs when the target means recent N items
- do not silently return an unauthenticated, challenge, captcha, or abnormal page as empty data

Viewer work:

- do not put input fields, dropdowns, or settings panels inside the widget body
- keep configuration in Kepo host settings through `widget.config.ts`
- render states clearly: ready, loading, empty, error, login-required, and any verified abnormal state
- keep remote images on the project-supported cached image path rather than bare remote URLs when the host provides cache support

Documentation work:

- `GUIDE.md` is user-facing; avoid provider, viewer, scraping, storage, and internal debug terms
- `VERSION.md` is release-facing; put the current version entry at the top before publishing

### 4. Dev Debug Loop

For ordinary plugin development, default to this local debugging path after edits:

```bash
pnpm typecheck
pnpm build
kp dev
```

Use `kp dev` as the live debugging handoff when the user needs to try the plugin in the Kepo client. Report that the dev loop is running, where logs are being tailed, and any immediate build/runtime errors. Do not stop a healthy dev loop by default; it is the expected hot-update session for the user to test the plugin. If follow-up edits are needed while it is still running, rely on the existing watcher unless the CLI state is broken or the user asks for a restart. If startup fails at the socket connection stage, explain that the local Kepo main app likely is not running or has lost `/tmp/kepo.sock`; ask the user to restart the main app, then rerun `kp dev`.

When `kp dev` is running, use provider event checks, stored data, and `.kepo/dev/log/output.log` as the evidence surface. Keep the dev loop alive while the user is testing or while hot updates are useful. Stop it only when the user asks, debugging is done and you are moving to install/publish, or the running watcher is in a bad state that requires a restart.

After the debug loop is clean, prompt the user with the next concrete choices:

- local deploy/install into the Kepo client: `kp local-publish` or `kp local-publish --force`
- remote backend review publish: `kp publish`

Do not automatically run local publish or remote publish after a normal development request unless the user asks for that release step.

### 5. Debug And Verify

Read [references/debug-and-release.md](references/debug-and-release.md) before debugging provider output, local installs, or publish failures.

For runtime bugs, default to this order:

1. Identify the exact `widgetId`.
2. Run `kp test-provider-event` for that widget.
3. Inspect `storagePreview`.
4. If preview truncates important data, inspect the stored payload or plugin logs.
5. Read `.kepo/dev/log/output.log` when present.
6. Patch provider or viewer.
7. Rerun the same provider event.
8. Run `pnpm typecheck` and `pnpm build` when the change touches code or release artifacts.
9. Start `kp dev` when the user needs to verify the fix in the local Kepo client; if a healthy `kp dev` watcher is already running, leave it running and let hot reload pick up the change.

Do not claim a provider bug is fixed from source reading alone. Report the command used, widget ID, stored item count, representative rows when relevant, and remaining limitations.

### 6. Local Install

Use local install when the user has finished debugging and wants to deploy/install the plugin into the Kepo client before backend publishing:

```bash
pnpm build
kp local-publish
```

If the same version is already installed or the user explicitly wants to overwrite local state:

```bash
kp local-publish --force
```

If socket connection fails, explain that the local Kepo client must be running for local install, provider event execution, or `kp dev`. If the client appears to be open but the socket is missing or disconnected, ask the user to restart the Kepo main app and retry. Do not replace this with a fake success.

After local install finishes, report the plugin ID, version, action, install path, and package path shown by the CLI.

### 7. Publish For Review

Before running `kp publish`, verify:

- `package.json` version is higher than the already published version when known
- `VERSION.md` top entry matches the package version
- `GUIDE.md` is user-facing and not empty
- `pnpm typecheck` passes
- `pnpm build` passes
- preview assets match manifest expectations when the plugin includes previews
- `kp login` has been completed or `kp publish` can obtain a saved token

When `kp publish` prompts for changelog, use a concise release summary that matches `VERSION.md`.

After publish, report:

- plugin ID
- version
- status
- review status
- any source archive or package path shown by the CLI

Do not describe a publish as completed unless the command actually returned success.

## User Guidance Standard

When the user sounds unsure, guide them with the next concrete action instead of explaining the entire system.

Good responses:

- "我先确认这是现有插件还是新插件，然后直接跑本地构建。"
- "这一步需要真实 widgetId，我会先从当前上下文和本地数据里找；找不到再问你。"
- "我会先跑 `kp dev` 让你在本地客户端调试；调试完以后你可以选本地部署或提交远程审核。"
- "发布会提交后台审核，我先做版本、说明和构建检查，最后再执行发布。"

Avoid:

- asking the user to choose a CLI command when the correct command is obvious
- asking the user to inspect logs manually
- telling the user to check whether the service is running without first trying the command and reading the error
- giving internal-only terms without plain-language explanation

## Validation Expectations

For code changes, run the smallest reliable validation:

- TypeScript/plugin change: `pnpm typecheck`
- ordinary plugin development handoff: `pnpm typecheck`, `pnpm build`, then `kp dev`
- build or release artifact change: `pnpm build`
- provider behavior change: `kp test-provider-event` with exact `widgetId`
- local client install: `kp local-publish` or `kp local-publish --force`
- publish: `kp publish` plus final status and review status

If validation is blocked by missing login, missing local client socket, missing widget ID, network, or permissions, state the blocker and what was already verified.

## References

Reference files are relative to this skill directory, not to the user's local project.

- CLI command behavior: [references/cli-commands.md](references/cli-commands.md)
- Public API usage: [references/api-usage.md](references/api-usage.md)
- Build Mode agent patterns: [references/build-mode-patterns.md](references/build-mode-patterns.md)
- Debugging and release runbook: [references/debug-and-release.md](references/debug-and-release.md)
