<!--
File purpose:
- Path: references/cli-commands.md
- Responsibility: Summarize the real kp CLI commands exposed by cli-v2 and how to use them during Kepo plugin development.
-->

# Kepo CLI Commands

Use this reference when choosing or explaining a `kp` command.

## Public `kp` Commands

The public `kp` binary registers these commands from `cli-v2/main.go`:

- `kp build`
- `kp publish`
- `kp local-publish`
- `kp dev`
- `kp test-provider-event`
- `kp login`
- `kp plugins list`
- `kp plugins get <pluginId>`
- `kp plugins versions <pluginId>`

`reviewd` has command code, but it is not registered in the public `kp` main entrypoint. Treat it as separate internal tooling unless the user explicitly asks about it.

The npm wrapper also accepts `npm-post-install`, but that is the package installation hook that downloads the platform binary. It is not a normal plugin development command.

## `kp build`

Run from a plugin root.

Behavior:

- builds Tailwind CSS when `src/tailwind.css` exists
- builds viewer bundle into `.kepo/prod/front.js`
- builds provider bundle into `.kepo/prod/provider.js`
- generates `.kepo/prod/manifest.json`
- copies assets and `GUIDE.md`
- with `--package` or `-p`, packages `.kepo/prod` output into `dist/<name>-<version>.kp`

Typical scaffold script:

```bash
pnpm build
```

The scaffold's `pnpm build` normally runs:

```bash
pnpm typecheck && kp build
```

Use direct packaging when needed:

```bash
kp build --package
kp build -p
```

## `kp dev`

Run from a plugin root.

Behavior:

- switches plugin ID to a dev ID
- builds into `.kepo/dev`
- watches the plugin directory except `.git`, `node_modules`, `dist`, editor folders, `.DS_Store`, and `.kepo`
- notifies the running Kepo client through local socket
- tails `.kepo/dev/log/output.log`

Use when the user wants interactive local development in the Kepo client.

Do not treat source files alone as proof that the running client has updated. When hot update matters, check `.kepo/dev` output or rerun `kp test-provider-event`.

## `kp test-provider-event`

Run from any directory as long as the local Kepo client is running and the target widget exists.

Required:

```bash
kp test-provider-event --widget-id <widgetId>
```

Common options:

```bash
kp test-provider-event --widget-id <widgetId> --event-name onScheduled
kp test-provider-event --widget-id <widgetId> --event-type action --event-name <actionName>
kp test-provider-event --widget-id <widgetId> --config key=value --config other='{"json":true}'
kp test-provider-event --widget-id <widgetId> --param key=value
kp test-provider-event --widget-id <widgetId> --persist-id <browserPersistId>
```

Defaults:

- `event-name`: `onScheduled`
- `event-type`: `lifecycle`

Output includes:

- success
- widgetId
- eventType
- eventName
- optional persistId
- error
- storagePreview

Use this before changing more code when a provider refresh result is wrong.

## `kp local-publish`

Run from a plugin root.

Behavior:

- runs the shared publish-artifact build flow without source archive
- generates version index
- builds production output
- packages `.kp`
- sends `localPublish` to the running Kepo client through socket

Use:

```bash
kp local-publish
```

Use force when reinstalling the same version or downgrading locally:

```bash
kp local-publish --force
```

This requires the local Kepo client socket to be available.

## `kp login`

Opens browser OAuth and saves a JWT token in the user's config directory under Kepo CLI auth state.

Use when publish or plugin listing fails because the saved auth token is missing or invalid.

## `kp publish`

Run from a plugin root.

Behavior:

- checks saved auth token
- prompts for a changelog
- generates `version-index.json` from `VERSION.md`
- builds production output
- validates manifest
- validates preview assets
- packages `.kp`
- creates a source archive
- uploads source archive to the server
- logs plugin ID, version, status, reviewStatus, package path, and source archive path

Use only when the user wants backend submission or review.

Before running:

- ensure package version is correct
- ensure `VERSION.md` top entry matches the version
- run `pnpm typecheck`
- run `pnpm build`
- prepare a changelog string of at least 10 characters

## `kp plugins`

Requires login.

Use to inspect backend plugin records:

```bash
kp plugins list --page 1 --page-size 20
kp plugins get <pluginId>
kp plugins versions <pluginId> --page 1 --page-size 20
```

Outputs JSON. Use this when checking already published versions, plugin status, or version history.

## Scaffold Commands

New plugin scaffolding is not a `kp` command. Use the package scaffold from the monorepo plugins directory:

```bash
cd /Users/levine/workspace/kepo-monorepo/plugins
pnpm create kepo-plugin <plugin-folder>
```

Do not hand-create scaffold baseline files when this command can create them.
