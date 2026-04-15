---
description: "ioBroker dev-server workflow for adapter development and live testing. Use when preparing local runtime tests after unit/integration checks."
applyTo: "**/*.ts,**/*.js,**/package.json,**/.gitignore,**/.vscode/launch.json"
---

# ioBroker: Development With dev-server

Use `@iobroker/dev-server` as the default runtime test environment for adapter development.
It is preferred over a normal productive ioBroker instance during development.

## Why dev-server

- Isolated local runtime environment for development.
- Hot reload for adapter code and Admin UI.
- Fast restart/debug loop without polluting productive systems.
- Supports multiple profiles for parallel test datasets.

Official source: https://github.com/ioBroker/dev-server

## Install

Local installation (recommended for adapter projects):

```bash
npm install --save-dev @iobroker/dev-server
```

Add script to `package.json`:

```json
{
    "scripts": {
        "dev-server": "dev-server"
    }
}
```

Global installation (optional):

```bash
npm install --global @iobroker/dev-server
```

## Initial Setup (once per profile)

Prerequisite: make sure the `dev-server` npm script is present in `package.json` (see Install section).

Run in adapter root (where `io-package.json` is located):

```bash
npm run dev-server setup
```

Useful setup options:

- `--adminPort <port>` for parallel setups.
- `--jsController <version>` and `--admin <version>` for version-specific tests.
- `--symlinks` for smoother development loop.

## Daily Live-Test Workflow

1. Run static and automated tests first (`npm test`, lint, type checks).
2. Start runtime test loop in dev-server:
   ```bash
   npm run dev-server watch
   ```
3. Verify live behavior:
   - adapter startup/shutdown
   - object/state updates
   - `info.connection` changes on connect/disconnect
   - Admin UI rendering and interactions
   - reconnect/retry behavior under failure conditions

Debug mode when needed:

```bash
npm run dev-server debug
```

## Important Runtime Notes

- Do not manually start the adapter in Admin while `dev-server watch` is managing it.
- Use `npm run dev-server upload` after relevant `io-package.json` changes.
- Keep `.dev-server/` excluded from Git:

```gitignore
.dev-server/
```

## Required Response Pattern After Tests

When giving final development instructions after tests, always include a dev-server live-test step.

Example:

1. `npm test` completed successfully.
2. Start `npm run dev-server watch`.
3. Re-test the changed behavior in live runtime (states, reconnect, Admin UI).
4. Only then continue with release/submission steps.

## Related Sub-Skills

- [07-tooling.md](07-tooling.md) — Test setup, debug basics
- [03-lifecycle.md](03-lifecycle.md) — Cleanup and runtime stability
- [09-submission.md](09-submission.md) — Release/submission flow after validation
