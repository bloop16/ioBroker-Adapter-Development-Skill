---
description: "ioBroker adapter tooling: error handling, logging, TypeScript/JavaScript configuration, ESLint, Prettier, and testing setup. Use when setting up build tools or writing logging/error handling code."
applyTo: "**/*.ts,**/*.js,**/tsconfig*.json,**/eslint.config*,**/package.json"
---

# ioBroker: Error Handling, Logging, TypeScript & Testing

## 15. Error Handling & Logging

### API / Network Error Handling

```typescript
try {
    const response = await axios.get(url, { timeout: 10000 });
    await this.setStateAsync("info.connection", true, true);
    return response.data;
} catch (error) {
    await this.setStateAsync("info.connection", false, true);
    if (axios.isAxiosError(error)) {
        if (error.response?.status === 404) {
            this.log.debug(`Not found: ${url}`);
            return null;
        }
        if (error.response?.status === 429) {
            this.log.warn(`Rate limited — will retry on next poll`);
            return null;
        }
        this.log.warn(`API request failed: ${error.message}`);
    } else {
        this.log.error(`Unexpected error fetching ${url}: ${error}`);
    }
    return null;
}
```

### Logging Rules

- **All log messages must be in English** (unless quoting an external source verbatim).
- **Never** use `console.log()` — always use `this.log.*`.

| Level | Use for |
|-------|---------|
| `this.log.silly()` | Raw data dumps, protocol messages, verbose tracing |
| `this.log.debug()` | Flow tracking, state changes, connection events |
| `this.log.info()` | Normal operations (started, connected, polling started) |
| `this.log.warn()` | Recoverable issues (timeout, rate limit, retry, config fallback) |
| `this.log.error()` | Failures needing user attention (auth failed, config invalid, unhandled exception) |

### Built-in Node.js Modules — Use `node:` Prefix [S5043]

```typescript
// ✅ CORRECT
import * as fs from "node:fs";
import * as path from "node:path";
import * as crypto from "node:crypto";
import { EventEmitter } from "node:events";

// ❌ WRONG — missing node: prefix
import * as fs from "fs";
```

---

## 16. Notifications & Sentry

### Notification System (user-facing alerts in Admin UI)

```json
// io-package.json
{
    "notifications": [{
        "scope": "myadapter",
        "name": {
            "en": "My Adapter",
            "de": "Mein Adapter"
        },
        "description": {
            "en": "Adapter notifications"
        },
        "categories": [{
            "category": "updates",
            "name": {
                "en": "Updates available",
                "de": "Updates verfügbar"
            },
            "severity": "notify",
            "limit": 1
        }]
    }]
}
```

**Important**: All notification texts must be translated into all 11 supported languages [E1112].  
See [04-admin-ui.md](04-admin-ui.md) for the full language list.

Usage in adapter code:
```typescript
await this.registerNotification("myadapter", "updates", "Device X has a firmware update available");
```

### Sentry Error Reporting (optional)

```json
// io-package.json
{
    "plugins": {
        "sentry": {
            "dsn": "https://your-dsn@sentry.iobroker.net/XX"
        }
    }
}
```

> `@iobroker/plugin-sentry` is **system-provided** — do **not** add it to `dependencies` in `package.json`.

---

## 18. TypeScript & JavaScript Configuration

### TypeScript Adapter — Three tsconfig Files

```
tsconfig.json          — Base config: IDE type checking for src/ + test/
tsconfig.build.json    — Production build: src/ → build/, excludes tests
tsconfig.check.json    — Extended checks: includes admin/
```

Typical `tsconfig.build.json`:
```json
{
    "extends": "./tsconfig.json",
    "compilerOptions": {
        "removeComments": true,
        "outDir": "build/"
    },
    "include": ["src/**/*.ts"],
    "exclude": ["src/**/*.test.ts"]
}
```

### JavaScript Adapter with Type Checking

```json
// tsconfig.json
{
    "compilerOptions": {
        "noEmit": true,
        "checkJs": true,
        "allowJs": true,
        "strict": true
    },
    "include": ["main.js", "lib/**/*.js"]
}
```

### Build Scripts in `package.json`

```json
{
    "scripts": {
        "build": "tsc -p tsconfig.build.json",
        "check": "tsc --noEmit -p tsconfig.check.json",
        "lint": "eslint -c eslint.config.mjs src/",
        "test": "npm run test:package && npm run test:unit",
        "test:package": "mocha test/package.js --exit",
        "test:unit": "mocha test/unit.js --exit",
        "test:integration": "mocha test/integration.js --exit",
        "translate": "translate-adapter",
        "release": "release-script"
    }
}
```

---

## 19. ESLint & Prettier

Use ioBroker's shared configs — do not create custom configs:

```javascript
// eslint.config.mjs
import config from "@iobroker/eslint-config";
export default [...config];
```

```javascript
// prettier.config.mjs
import prettierConfig from "@iobroker/eslint-config/prettier.config.mjs";
export default { ...prettierConfig };
```

---

## 20. Testing

### Standard Framework: `@iobroker/testing` + Mocha

```javascript
// test/package.js — validates io-package.json and package.json structure
const path = require("node:path");
const { tests } = require("@iobroker/testing");
tests.packageFiles(path.join(__dirname, ".."));
```

```javascript
// test/unit.js — unit tests
const path = require("node:path");
const { tests } = require("@iobroker/testing");
tests.unit(path.join(__dirname, ".."), {
    allowedExitCodes: [11],
});
```

```javascript
// test/integration.js — integration tests (starts/stops adapter)
const path = require("node:path");
const { tests } = require("@iobroker/testing");
tests.integration(path.join(__dirname, ".."), {
    allowedExitCodes: [11],
});
```

### GitHub Actions Requirements

- **Must** use the standard `test-and-release.yml` workflow from the Adapter Creator.
- The `deploy` job must depend on `check-and-lint` and `adapter-tests` jobs [E3016].

**If using trusted publishing (`ioBroker/testing-action-deploy@v1`)**:
- Set job-level permissions: `contents: write` and `id-token: write` [W3018]
- Do **not** specify the `npm-token` parameter (incompatible with trusted publishing) [W3019]

```yaml
jobs:
  deploy:
    needs: [check-and-lint, adapter-tests]
    permissions:
      contents: write
      id-token: write
    steps:
      - uses: ioBroker/testing-action-deploy@v1
        # Do NOT add: npm-token: ${{ secrets.NPM_TOKEN }}
```

### `@iobroker/testing` Dependency

- **Must** be in `devDependencies` — never in `dependencies` [repochecker Error]

---

## Debugging Tips

When starting manually for debugging:

```bash
# Start adapter instance 0 at debug level, force start even if disabled
node main.js 0 debug --force

# Start even without existing configuration (useful for install scripts)
node main.js 0 info --install
```

VS Code launch configuration:
```json
{
    "type": "node",
    "request": "launch",
    "name": "Debug Adapter",
    "program": "${workspaceFolder}/build/main.js",
    "args": ["0", "debug", "--force"],
    "sourceMaps": true
}
```

---

## Related Sub-Skills

- [03-lifecycle.md](03-lifecycle.md) — Lifecycle and cleanup
- [06-patterns.md](06-patterns.md) — Error handling in polling
- [08-packaging.md](08-packaging.md) — package.json dependency rules
