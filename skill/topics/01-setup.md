---
description: "ioBroker adapter setup and naming conventions. Use when starting a new adapter or reviewing its naming."
applyTo: "**/io-package.json,**/package.json,**/*.ts,**/*.js"
---

# ioBroker: Project Setup & Naming Conventions

## 1. New Adapter — NEVER Generate From Scratch

- **Always** use the official Adapter Creator for new adapters:
  ```bash
  npx @iobroker/create-adapter@latest
  ```
- **Never** copy another adapter and modify it.
- **Never** generate a full adapter scaffold from memory — AI models have outdated knowledge of ioBroker project structures.
- The Creator produces the correct `io-package.json`, GitHub Actions, Admin UI skeleton, ESLint config, and testing setup.
- **Never** copy `io-package.json` from an installed adapter — installed files contain `installedFrom`, `installedVersion` artifacts that must not be committed.

---

## 2. Naming Conventions

### Repository & Package Naming

| Item | Convention | Example |
|------|-----------|---------|
| GitHub repository | `ioBroker.<adapterName>` (**capital B**) | `ioBroker.shelly` |
| npm package | `iobroker.<adaptername>` (all **lowercase**) | `iobroker.shelly` |
| Adapter name | lowercase only, `a-z0-9` and `-`; no `_` at start | `my-adapter` |

> npm does not allow uppercase letters in package names — the lowercase rule is enforced by npm itself.

### Object ID Rules

| Rule | Detail | Good | Bad |
|------|--------|------|-----|
| Allowed characters | `A-Za-z0-9`, `-`, `_`, `.` | `devices.sensor1.temp` | `Nächstes Rennen` |
| Dot as separator | Each `.` is a hierarchy level | `device.channel.state` | — |
| No spaces | Spaces are rejected | — | `my device` |
| No umlauts/special chars | ASCII only | — | `straße` |

### ID Sanitization — Always Sanitize External Input

Any identifier coming from external data (API responses, device names, user input) must be sanitized before use as an object ID:

```typescript
// Sanitize to allowed characters
const safeId = externalName.replace(/[^a-zA-Z0-9_-]/g, '_');

// Additional: ensure it doesn't start with underscore
const finalId = safeId.replace(/^_+/, '') || 'device';
```

### Port & IP Attribute Names (repochecker enforced)

- The config attribute for a TCP/UDP port **must** be named `port`
- The config attribute for an IP address **must** be named `ip`
- The config attribute for a bind address **may** be named `bind` (or `v6bind` for IPv6)

---

## Related Sub-Skills

- [02-objects-states.md](02-objects-states.md) — Object hierarchy and state definitions
- [08-packaging.md](08-packaging.md) — io-package.json structure, package.json requirements
- [09-submission.md](09-submission.md) — Repository submission workflow
