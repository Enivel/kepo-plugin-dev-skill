<!--
File purpose:
- Path: references/debug-and-release.md
- Responsibility: Provide the practical runbook for Kepo plugin runtime debugging, local installation, and backend publish/review.
-->

# Debugging And Release Runbook

## Provider Debugging

When provider output is wrong, prove the runtime result before patching broadly.

Default sequence:

1. Find the exact `widgetId`.
2. Run `kp test-provider-event --widget-id <widgetId> --event-name onScheduled`.
3. Read `success`, `error`, and `storagePreview`.
4. If arrays or nested objects are truncated, inspect the real stored data or plugin logs.
5. Read `.kepo/dev/log/output.log` when present.
6. Patch the provider.
7. Rerun the same provider event.
8. Report the tested widget ID and final stored data evidence.

Do not choose a widget by plugin name, visual order, or size. Use a real widget instance ID from user context, local data, or the created dev widget.

## Logs To Add Temporarily

For provider extraction bugs, add logs that explain the decision, not just the final failure.

Useful fields:

- widget ID
- config summary
- source URL or endpoint
- acquisition method
- response status or readiness marker
- parsed count
- stored count
- stop reason
- rejection reason for skipped candidates

For list-like widgets, include:

- first-screen count
- per-scroll or per-page new count
- duplicate count when relevant
- final count
- stop reason such as target reached, page bottom, API no more data, repeated no-new-record rounds, or blocked by challenge page

Remove or reduce noisy candidate logs after the bug is understood unless ongoing diagnostics need them.

## Hot Update Checks

When the user says the dev run hot-updated, verify the running output if the behavior still looks old.

Check:

- the TypeScript source changed
- `.kepo/dev/provider.js` or other generated dev output contains the new marker or logic when available
- `kp test-provider-event` has been rerun after the update
- `.kepo/dev/log/output.log` lines are from the rerun, not an earlier refresh

Do not argue from source code alone when runtime output still shows old behavior.

## Viewer Debugging

Verify viewer changes against real stored payload shape.

Check:

- every configured widget size renders intentionally
- loading, empty, error, login-required, and ready states are present when applicable
- config controls are not inside the widget body
- scrollable lists render the intended stored item count unless the design explicitly says preview-only
- images render through the host-supported cached image path when needed

Do not hide source data loss in the viewer. If provider output is missing records, fix provider output first.

## Local Install

Use local install after build when the user wants to try the plugin in the Kepo app:

```bash
pnpm build
kp local-publish
```

Use:

```bash
kp local-publish --force
```

when reinstalling the same version, downgrading, or replacing an existing local plugin build.

If local publish fails with socket errors, the local Kepo client is not reachable. Report that direct cause and do not claim install success.

## Release Preparation

Before backend publish:

- bump `package.json` version when the version already exists
- update `VERSION.md` with the current version at the top
- keep `GUIDE.md` user-facing
- run `pnpm install` if dependencies changed
- run `pnpm typecheck`
- run `pnpm build`
- check preview assets if the plugin manifest expects them
- run `kp plugins versions <pluginId>` when the already-published version is uncertain

`kp publish` will generate `version-index.json`. If the file changes, keep it with the release diff.

## Changelog Input

`kp publish` prompts for a changelog and requires at least 10 characters.

Use one concise sentence based on the top `VERSION.md` entry, for example:

```text
Fix feed refresh and improve login-required state handling.
```

Do not invent release notes that are not in the code.

## Publish Confirmation

After `kp publish` succeeds, report:

- plugin ID
- version
- status
- review status
- package path
- source archive path

If the user asked to submit for backend review, the important result is the backend `reviewStatus`, not merely local build success.

## Failure Handling

For build failures:

- read the first real TypeScript or bundler error
- patch the responsible plugin file
- rerun the same command

For publish failures:

- distinguish auth failure, duplicate version, build failure, preview validation failure, network/API failure, and server rejection
- do not bump versions repeatedly without understanding whether a previous publish actually succeeded
- use `kp plugins versions <pluginId>` when duplicate-version state is unclear

For provider failures:

- distinguish no runtime execution, authenticated request mismatch, abnormal page, parse failure, storage write issue, and viewer render issue
- avoid returning empty data for challenge, captcha, login shell, or blocked responses
