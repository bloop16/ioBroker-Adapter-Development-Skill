---
description: "Universal ioBroker adapter development skill. Use when creating, modifying, debugging, or reviewing any ioBroker adapter. Enforces correct object hierarchy, state roles, Admin UI (JSONConfig), lifecycle management, testing, repochecker compliance, and submission requirements."
agent: "agent"
---

# ioBroker Adapter Development Skill

You are an expert ioBroker adapter developer. Follow these rules **strictly** — they are enforced by the ioBroker adapter checker (repochecker) and the repository review process.

**Primary source of truth**: https://github.com/Jey-Cee/iobroker-ai-developer-guide
**Validation tool**: `npx @iobroker/repochecker <repo-url>`

---

## 1. Project Setup — NEVER Generate From Scratch

- **Always** use the official Adapter Creator for new projects:
  ```bash
  npx @iobroker/create-adapter@latest
  ```
- **Never** copy another adapter and modify it.
- **Never** generate a full adapter scaffold from memory — AI models have outdated knowledge of ioBroker project structures.
- The Creator produces the correct `io-package.json`, GitHub Actions, Admin UI skeleton, ESLint config, and testing setup.

---

## 2. Naming Conventions

| Item | Convention | Example |
|------|-----------|---------|
| GitHub repo | `ioBroker.<adapterName>` (capital B) | `ioBroker.shelly` |
| npm package | `iobroker.<adaptername>` (all lowercase) | `iobroker.shelly` |
| Adapter name | lowercase, only `a-z0-9`, `-`; no `_` at start | `my-adapter` |
| Object IDs | Only `A-Za-z0-9`, `-`, `_`, `.` (dot as level separator) | `devices.sensor1.temperature` |
| Object IDs | **No** spaces, umlauts, or special characters | ❌ `Nächstes Rennen` |
| ID sanitization | Always sanitize external input before using as IDs | `input.replace(/[^a-zA-Z0-9_-]/g, '_')` |

---

## 3. Object Hierarchy — Critical

### Mandatory Structure: `device → channel → state`

```
adapter.0.myDevice                          // type: "device"
adapter.0.myDevice.sensors                  // type: "channel"
adapter.0.myDevice.sensors.temperature      // type: "state"
```

### Rules

1. **All intermediate objects MUST be explicitly created.** For path `a.b.c`, create objects for `a`, `b`, AND `c`. The ioBroker object tree does NOT auto-create parents.
2. **States NEVER nest under other states** — always use device/channel/state hierarchy.
3. If the adapter is simple (no physical devices), channels are still recommended for grouping related states.
4. Use `type: "folder"` for organizational grouping when device/channel doesn't fit semantically.
5. Use `type: "meta"` with `common.type: "meta.user"` for file storage areas.

### Object Creation — Always Use Safe Methods

```typescript
// ✅ CORRECT — won't overwrite user customizations
await this.setObjectNotExistsAsync("sensors", {
    type: "channel",
    common: { name: "Sensors" },
    native: {},
});

await this.setObjectNotExistsAsync("sensors.temperature", {
    type: "state",
    common: {
        name: "Temperature",
        type: "number",
        role: "value.temperature",
        unit: "°C",
        read: true,
        write: false,
    },
    native: {},
});

// ❌ WRONG — setObject overwrites existing objects including user customizations!
await this.setObjectAsync("sensors.temperature", { ... });
```

### Recommended Helper Pattern

Keep object creation DRY with helper functions:

```typescript
async function ensureChannel(adapter: ioBroker.Adapter, id: string, name: string | ioBroker.StringOrTranslated): Promise<void> {
    await adapter.setObjectNotExistsAsync(id, { type: "channel", common: { name }, native: {} });
}

async function ensureFolder(adapter: ioBroker.Adapter, id: string, name: string | ioBroker.StringOrTranslated): Promise<void> {
    await adapter.setObjectNotExistsAsync(id, { type: "folder", common: { name }, native: {} });
}

async function ensureState(
    adapter: ioBroker.Adapter,
    id: string,
    common: Partial<ioBroker.StateCommon>,
    value?: ioBroker.StateValue,
): Promise<void> {
    await adapter.setObjectNotExistsAsync(id, {
        type: "state",
        common: { read: true, write: false, ...common } as ioBroker.StateCommon,
        native: {},
    });
    if (value !== undefined) {
        await adapter.setStateChangedAsync(id, value, true);
    }
}
```

### Object API Summary

| Method | Use When |
|--------|----------|
| `setObjectNotExistsAsync()` | Creating objects during init — won't overwrite |
| `extendObjectAsync()` | Updating existing objects — merges, doesn't replace |
| `setStateChangedAsync()` | Setting state only if value actually changed — reduces DB writes |
| `setObjectAsync()` | **Avoid** — overwrites everything including user customizations |

---

## 4. State Definitions — Mandatory Fields

Every state **must** have these `common` attributes:

```typescript
{
    type: "state",
    common: {
        name: "Human readable name",     // REQUIRED — string or { en: "...", de: "..." }
        type: "number",                   // REQUIRED: "string" | "number" | "boolean" | "array" | "object" | "json" | "mixed" | "file"
        role: "value.temperature",        // REQUIRED: must be specific (see Section 5)
        read: true,                       // REQUIRED
        write: false,                     // REQUIRED
        unit: "°C",                       // Recommended for numbers
        min: -50,                         // Recommended for numbers
        max: 100,                         // Recommended for numbers
        def: 0,                           // Recommended: default value
    },
    native: {},
}
```

### Notes

- `"array"`, `"object"`, `"mixed"`, and `"file"` values must be serialized via `JSON.stringify()`.
- Use `role: "json"` with `type: "string"` for structured JSON data stored as a string.
- Use `common.states` for finite enum sets:
  ```typescript
  common: { type: "number", role: "value", states: { 0: "Offline", 1: "Online", 2: "Busy" } }
  ```

---

## 5. State Roles — NEVER Use Generic "state"

The role `"state"` is **rejected** in adapter reviews. Always use the most specific role.

### Value Roles (read-only numbers)

| Role | Description | Unit |
|------|-------------|------|
| `value` | Generic numeric value | — |
| `value.temperature` | Temperature | °C / °F |
| `value.humidity` | Humidity | % |
| `value.pressure` | Pressure | hPa |
| `value.speed` | Speed | km/h |
| `value.time` | Time (epoch or seconds) | s / ms |
| `value.interval` | Interval | sec |
| `value.voltage` | Voltage | V |
| `value.current` | Current | A / mA |
| `value.power` | Power | W / kW |
| `value.energy` | Energy | Wh / kWh |
| `value.battery` | Battery level | % |
| `value.brightness` | Brightness | lux |
| `value.gps.latitude` | GPS latitude | ° |
| `value.gps.longitude` | GPS longitude | ° |
| `value.fill` | Fill level | % / l |

### Level Roles (writable numbers)

| Role | Description |
|------|-------------|
| `level` | Generic writable level |
| `level.volume` | Volume (0–100) |
| `level.dimmer` | Dimmer brightness (0–100) |
| `level.temperature` | Target temperature setpoint |
| `level.blind` | Blind/shutter position |
| `level.color.red/green/blue` | Color channels (0–255) |
| `level.color.hue` | Hue (0–360) |
| `level.color.saturation` | Saturation (0–100) |
| `level.color.temperature` | Color temperature (K) |

### Indicator Roles (read-only booleans)

| Role | Description |
|------|-------------|
| `indicator` | Generic boolean indicator |
| `indicator.connected` | Device/service connectivity |
| `indicator.reachable` | Device reachable on network |
| `indicator.maintenance` | Maintenance required |
| `indicator.maintenance.unreach` | Device unreachable |
| `indicator.lowbat` | Low battery |
| `indicator.alarm` | Generic alarm |
| `indicator.alarm.fire` | Fire alarm |
| `indicator.alarm.flood` | Flood alarm |
| `indicator.working` | Operation in progress |

### Switch Roles (writable booleans)

| Role | Description |
|------|-------------|
| `switch` | Generic on/off switch |
| `switch.light` | Light switch |
| `switch.power` | Power switch |
| `switch.lock` | Lock control |
| `switch.boost` | Boost mode |
| `switch.enable` | Enable/disable feature |

### Button Roles (write-only triggers)

| Role | Notes |
|------|-------|
| `button` | `read: false, write: true` |
| `button.start` / `button.stop` | Action triggers |
| `button.open` / `button.close` | Door/gate triggers |

### Text & Data Roles

| Role | Description |
|------|-------------|
| `text` | Generic text value |
| `text.url` | URL |
| `json` | JSON-encoded data (`type: "string"`) |
| `html` | HTML content |
| `date` | ISO date string |
| `list` | List/array value |
| `info.name` | Device/adapter name |
| `info.address` | Address (IP, MAC) |
| `info.status` | Status text |
| `info.firmware` | Firmware version |
| `info.serial` | Serial number |

### Channel Roles (optional but recommended)

Channels can have roles to describe device functionality:
`sensor`, `sensor.door`, `sensor.window`, `light.switch`, `light.dimmer`, `light.color`, `blind`, `thermo`, `media`, `media.music`, `info`, `alarm`, `phone`

### Rules

- `value.*` → read-only numbers; `level.*` → writable numbers
- `indicator.*` → read-only booleans; `switch.*` → writable booleans
- `button.*` → write-only triggers (`read: false, write: true`)
- `text.*` → read-only strings; `json` → JSON-encoded string data
- When no specific subrole fits, use the category root (`value`, `level`, `indicator`, `switch`, `text`)

---

## 6. The `ack` Flag — Critical Concept

The `ack` flag distinguishes **confirmed device values** from **user commands**:

```typescript
// ack=true → "This is the actual confirmed value from the device/API"
await this.setStateAsync("temperature", { val: 22.5, ack: true });

// ack=false → "User/script wants to set this value" (a command)
```

### Handling in onStateChange

```typescript
async onStateChange(id: string, state: ioBroker.State | null | undefined): Promise<void> {
    if (!state || state.ack) {
        return; // Ignore deletions and already-confirmed values
    }
    // state.ack === false → command from user/script
    try {
        await this.sendCommandToDevice(id, state.val);
        await this.setStateAsync(id, { val: state.val, ack: true }); // Confirm
    } catch (err) {
        this.log.error(`Failed to execute command for ${id}: ${err}`);
    }
}
```

### Rules

- **All values FROM the device/API** → always `ack: true`
- **User commands TO the device** → received with `ack: false`, confirmed with `ack: true` after execution
- **Never ignore the ack flag** — always check `!state.ack` in `onStateChange`
- Some protocol bridges may process both ack states; this requires explicit configuration

---

## 7. Adapter Lifecycle & Timer Management

### Use Adapter Methods — NEVER Native Node.js

```typescript
// ✅ CORRECT — adapter-managed timers (compact-mode safe)
this.myInterval = this.setInterval(() => this.poll(), 60000);
this.myTimeout = this.setTimeout(() => this.reconnect(), 5000);

// ❌ WRONG — native timers leak in compact mode
setInterval(() => this.poll(), 60000);
setTimeout(() => this.reconnect(), 5000);
```

### Process Termination

```typescript
// ✅ CORRECT
this.terminate("Reason for termination");

// ❌ WRONG — kills the entire process, breaks compact mode
process.exit(0);
```

### Unload — Clean Up EVERYTHING

```typescript
private onUnload(callback: () => void): void {
    try {
        this.isShuttingDown = true;
        this.setState("info.connection", false, true);

        // Clear all timers
        if (this.pollingInterval) { this.clearInterval(this.pollingInterval); this.pollingInterval = null; }
        if (this.reconnectTimeout) { this.clearTimeout(this.reconnectTimeout); this.reconnectTimeout = null; }

        // Close all connections (WebSocket, HTTP, MQTT, TCP, etc.)
        if (this.connection) { this.connection.close(); this.connection = null; }
        if (this.server) { this.server.destroy(); this.server = null; }

        // Reject pending request queues
        this.pendingRequests.forEach(r => r.reject(new Error("Shutting down")));
        this.pendingRequests = [];
    } catch (e) {
        // Ignore errors during cleanup
    } finally {
        callback(); // MUST always be called
    }
}
```

### Graceful Shutdown Pattern (for adapters with active requests)

```typescript
async onUnload(callback: () => void): Promise<void> {
    this.isShuttingDown = true;
    const forceTimer = setTimeout(() => callback(), 2000); // Force after 2s
    try {
        if (this.activeRequests > 0) {
            await this.waitForPendingRequests(1500);
        }
    } finally {
        clearTimeout(forceTimer);
        callback();
    }
}
```

### Rules

- **Every** timer, interval, WebSocket, HTTP client, TCP server, and file handle must be cleaned up in `onUnload`.
- The `callback()` **must always be called**, even on errors (use `finally` or force timeout).
- Use an `isShuttingDown` flag to prevent new operations after unload begins.
- Test with **compact mode**: Start → Run → Stop → Restart must work cleanly.

---

## 8. Connection State (`info.connection`)

**Required** for all adapters that connect to external services, devices, or protocols.

### Declaration in io-package.json

```json
"instanceObjects": [
    {
        "_id": "info",
        "type": "channel",
        "common": { "name": "Information" }
    },
    {
        "_id": "info.connection",
        "type": "state",
        "common": {
            "name": "Device or service connected",
            "role": "indicator.connected",
            "type": "boolean",
            "read": true,
            "write": false,
            "def": false
        },
        "native": {}
    }
]
```

### Usage

```typescript
await this.setStateAsync("info.connection", true, true);   // Connected
await this.setStateAsync("info.connection", false, true);  // Disconnected
```

### For Multi-Device Adapters

Track per-device status and derive global state:

```typescript
this.onlineDevices[deviceId] = isAlive;
const anyOnline = Object.values(this.onlineDevices).some(v => v);
await this.setStateAsync("info.connection", anyOnline, true);
```

Shows as green/yellow/red indicator in Admin UI. Set to `false` on startup and during unload.

---

## 9. Admin UI — JSONConfig

### Always Use JSONConfig

- **Never** generate `admin/index_m.html` or `admin/index.html` (deprecated Admin2/Materialize).
- Use `jsonConfig.json` or `jsonConfig.json5`.
- Set in `io-package.json`: `"adminUI": { "config": "json" }`

### Basic Panel Structure

```json
{
    "type": "panel",
    "i18n": true,
    "items": {
        "host": { "type": "text", "label": "IP Address", "sm": 12, "md": 6 },
        "port": { "type": "number", "label": "Port", "sm": 12, "md": 6, "min": 1, "max": 65535 },
        "pollingInterval": { "type": "number", "label": "Polling Interval (s)", "sm": 12, "md": 6, "min": 5, "max": 86400 }
    }
}
```

### Tab-Based Structure (for complex configs)

```json
{
    "type": "tabs",
    "i18n": true,
    "items": {
        "connectionTab": { "type": "panel", "label": "Connection", "items": { ... } },
        "advancedTab": { "type": "panel", "label": "Advanced", "items": { ... } }
    }
}
```

### Common Field Types

| Type | Use for |
|------|---------|
| `text` | Host, username, API path |
| `number` | Port, interval, timeout (with min/max) |
| `checkbox` | Enable/disable features |
| `select` | Protocol selection, modes, dropdown choices |
| `ip` | Bind address |
| `password` | API keys, passwords (auto-masked) |
| `color` | UI customization |
| `chips` | Multi-value tags (topics, patterns) |
| `instance` | Adapter instance selector |
| `table` | Editable lists (devices, mappings) |
| `sendTo` | Button executing adapter message |
| `header` / `staticText` | Labels and info text |

### Conditional Visibility & Responsive Layout

```json
{
    "serverPort": {
        "type": "number",
        "label": "Server Port",
        "hidden": "data.mode !== 'server'",
        "disabled": "!data.enableServer",
        "sm": 12, "md": 6, "lg": 4
    }
}
```

### Naming Rules (repochecker enforced)

- Port attribute **must** be named `port`
- IP attribute **must** be named `ip`
- Do **not** use `admin/words.js` in new adapters — it is considered outdated when jsonConfig is used [W5039]
- Remove obsolete files `admin/index.html`, `admin/index_m.html`, `admin/style.css` when using jsonConfig [W5047]
- If `admin/jsonConfig.json` is present, `common.adminUI.config` **must** be set to `"json"` in `io-package.json` [W5046]
- If `native` has config properties but `adminUI.config` is `"none"`, the config will be unusable [W1113]
- Do **not** use deprecated methods: `createState`, `createChannel`, `createDevice`, `deleteState`, `deleteChannel`, `deleteDevice` — use `setObjectNotExistsAsync` / `delObjectAsync` instead [W5033]
- jsonConfig files are validated against the official JSON schema automatically [E5512]; ensure your config is schema-compliant

---

## 10. Translations (i18n)

### Standard Directory Structure

```
admin/
├── jsonConfig.json
├── i18n/
│   ├── en.json          ← MANDATORY
│   ├── de.json          ← Required by repochecker
│   ├── ru.json, pt.json, nl.json, fr.json
│   ├── it.json, es.json, pl.json, uk.json, zh-cn.json
```

### Format (flat key-value)

```json
{ "Polling Interval": "Polling Interval", "IP Address": "IP Address" }
```

### Rules

- Set `"i18n": true` in jsonConfig.json.
- **English is mandatory**; `de` is required; all others strongly recommended.
- Use the ioBroker Translator: https://translator.iobroker.in/
- **Do not** hardcode language strings directly in jsonConfig.json.
- i18n directories **must be included** in the npm package — check `package.json` `"files"` field or `.npmignore` [E9506, E9507].
- Do not leave example keys (`option1`, `option2`) in translations [W5041].

---

## 11. Passwords & Secrets

**Never** store passwords or API keys in plaintext.

```json
{
    "encryptedNative": ["password", "apiKey"],
    "protectedNative": ["password", "apiKey"]
}
```

| Field | Purpose |
|-------|---------|
| `encryptedNative` | Encrypted at rest in the database |
| `protectedNative` | Hidden from API responses and admin UI network traffic |

- Always list sensitive fields in **both** arrays.
- Every key in `encryptedNative`/`protectedNative` must also exist in `native`.
- The adapter receives decrypted values at runtime via `this.config.*`.
- The repochecker warns if suspicious keys (containing "pass", "key", "secret", "token") are not protected.

---

## 12. Scheduling & Polling

### Simple Intervals — Use Adapter Timers

```typescript
this.pollingInterval = this.setInterval(() => this.fetchData(), this.config.pollingInterval * 1000);
```

### Anti-Thundering-Herd: Add Jitter

All instances polling at the same second can DDoS external APIs:

```typescript
const baseMs = this.config.pollingInterval * 1000;
const jitter = Math.floor(Math.random() * 5000);
this.setTimeout(() => {
    this.pollingInterval = this.setInterval(() => this.fetchData(), baseMs);
}, jitter);
```

### DON'T Use Scheduling Libraries for Simple Intervals

- Avoid `node-cron`, `node-schedule` for basic periodic polling.
- Exception: Complex cron patterns (astronomical events, business hours) may justify scheduling libraries.

### Request Throttling Pattern

For external APIs, limit concurrent requests:

```typescript
private activeRequests = 0;
private readonly maxConcurrent = 3;
private queue: Array<{ resolve: Function; reject: Function }> = [];

private async throttled<T>(fn: () => Promise<T>): Promise<T> {
    if (this.activeRequests >= this.maxConcurrent) {
        await new Promise<void>((resolve, reject) => this.queue.push({ resolve, reject }));
    }
    this.activeRequests++;
    try { return await fn(); }
    finally {
        this.activeRequests--;
        this.queue.shift()?.resolve();
    }
}
```

---

## 13. Message Handling (`messagebox` / `sendTo`)

For adapters that need to receive commands from other adapters, scripts, or the admin UI.

### Enable in io-package.json

```json
{ "common": { "messagebox": true } }
```

### Handle Messages

```typescript
constructor(options: Partial<utils.AdapterOptions> = {}) {
    super({ ...options, name: "myadapter" });
    this.on("message", this.onMessage.bind(this));
}

private async onMessage(obj: ioBroker.Message): Promise<void> {
    if (!obj?.command) return;
    switch (obj.command) {
        case "testConnection":
            try {
                await this.testConnection(obj.message);
                this.sendTo(obj.from, obj.command, { success: true }, obj.callback);
            } catch (err: any) {
                this.sendTo(obj.from, obj.command, { success: false, error: err.message }, obj.callback);
            }
            break;
        default:
            this.log.warn(`Unknown command: ${obj.command}`);
    }
}
```

### Calling from Admin UI (jsonConfig.json)

```json
{
    "testBtn": {
        "type": "sendTo",
        "label": "Test Connection",
        "command": "testConnection",
        "jsonData": "{\"host\": \"${data.host}\", \"port\": ${data.port}}",
        "showProcess": true
    }
}
```

### Calling from Scripts or Other Adapters

```javascript
sendTo("myadapter.0", "testConnection", { host: "192.168.1.1" }, (result) => {
    console.log(result.success ? "OK" : result.error);
});
```

---

## 14. Compact Mode

All adapters should support compact mode:

```json
// io-package.json
{ "common": { "compact": true } }
```

### Entry Point Pattern

```typescript
// TypeScript
if (require.main !== module) {
    module.exports = (options: Partial<utils.AdapterOptions> | undefined) => new MyAdapter(options);
} else {
    (() => new MyAdapter())();
}
```

```javascript
// JavaScript
if (require.main !== module) {
    module.exports = (options) => new MyAdapter(options);
} else {
    new MyAdapter();
}
```

### Compact Mode Checklist

- ✅ `this.setTimeout` / `this.setInterval` (not native)
- ✅ `this.terminate()` (not `process.exit()`)
- ✅ Clean up ALL resources in `onUnload`
- ✅ No global mutable state shared between instances
- ✅ Start → Run → Stop → Restart cycle works

---

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
        if (error.response?.status === 404) { this.log.debug(`Not found: ${url}`); return null; }
        if (error.response?.status === 429) { this.log.warn(`Rate limited`); return null; }
        this.log.warn(`API failed: ${error.message}`);
    } else {
        this.log.error(`Unexpected error: ${error}`);
    }
    return null;
}
```

### Logging Rules

- **All logs must be in English** (unless quoting an external source verbatim).
- **Never** use `console.log()` — always `this.log.*`.

| Level | Use for |
|-------|---------|
| `this.log.silly()` | Raw data dumps, protocol messages |
| `this.log.debug()` | Flow tracking, state changes |
| `this.log.info()` | Normal operations (connected, started polling) |
| `this.log.warn()` | Recoverable issues (timeout, rate limit, retry) |
| `this.log.error()` | Failures needing attention (auth failed, config invalid) |

---

## 16. Notifications & Sentry

### Notification System (for user-facing alerts)

```json
// io-package.json
{
    "notifications": [{
        "scope": "myadapter",
        "name": { "en": "My Adapter", "de": "Mein Adapter" },
        "description": { "en": "Adapter notifications" },
        "categories": [{
            "category": "updates",
            "name": { "en": "Updates available", "de": "Updates verfügbar" },
            "severity": "notify",
            "limit": 1
        }]
    }]
}
```

**Important**: Notifications must be translated into **all** supported languages (en, de, ru, pt, nl, fr, it, es, pl, uk, zh-cn) [E1112].

Usage: `await this.registerNotification("myadapter", "updates", "Device X has firmware update");`

### Sentry Error Reporting (optional)

```json
{ "plugins": { "sentry": { "dsn": "https://your-dsn@sentry.iobroker.net/XX" } } }
```

Note: `@iobroker/plugin-sentry` is a system-provided package — do **not** add it to `dependencies`.

---

## 17. Device Manager Integration (optional)

For adapters managing multiple physical devices:

```json
// io-package.json
{
    "common": { "supportedMessages": { "deviceManager": true } },
    "globalDependencies": [{ "admin": ">=7.8.19" }]
}
```

Use `@iobroker/dm-utils` for standardized device management UI in admin.

---

## 18. TypeScript & JavaScript Configuration

### TypeScript Adapter (3 tsconfig files)

```
tsconfig.json          — Base config (IDE, type checking for src/ + test/)
tsconfig.build.json    — Production build (src/ → build/, excludes tests)
tsconfig.check.json    — Extended checks (includes admin/)
```

### JavaScript Adapter with Type Checking

```json
// tsconfig.json
{ "compilerOptions": { "noEmit": true, "checkJs": true, "allowJs": true }, "include": ["main.js", "lib/**/*.js"] }
```

### Build Scripts

```json
{
    "build": "tsc -p tsconfig.build.json",
    "check": "tsc --noEmit -p tsconfig.check.json",
    "lint": "eslint -c eslint.config.mjs src/",
    "test": "mocha --exit",
    "translate": "translate-adapter",
    "release": "release-script"
}
```

---

## 19. ESLint & Prettier

Use ioBroker's shared configs:

```javascript
// eslint.config.mjs
import config from "@iobroker/eslint-config";
export default [...config];

// prettier.config.mjs
import prettierConfig from "@iobroker/eslint-config/prettier.config.mjs";
export default { ...prettierConfig };
```

---

## 20. Testing

### Standard Framework: `@iobroker/testing` + Mocha

```javascript
// test/package.js — Package validation (io-package.json, package.json)
const { tests } = require("@iobroker/testing");
tests.packageFiles(path.join(__dirname, ".."));

// test/unit.js — Unit tests
tests.unit(path.join(__dirname, ".."));

// test/integration.js — Integration tests (startup/shutdown)
tests.integration(path.join(__dirname, ".."));
```

### Scripts

```json
{
    "test:package": "mocha test/package.js --exit",
    "test:unit": "mocha test/unit.js --exit",
    "test:integration": "mocha test/integration.js --exit",
    "test": "npm run test:package && npm run test:unit"
}
```

### GitHub Actions

- **Must** use the standard `test-and-release.yml` workflow from the Adapter Creator.
- The `deploy` job must depend on `check-and-lint` and `adapter-tests` (or their equivalents) [E3016].
- If using `ioBroker/testing-action-deploy@v1` (trusted publishing), ensure:
  - Job-level permissions `contents: write` and `id-token: write` are set [W3018]
  - The `npm-token` parameter is **not** specified (incompatible with trusted publishing) [W3019]
- Additional custom tests are fine, but **never remove** the standard ones.

---

## 21. io-package.json — Complete Reference

### Mandatory Common Fields

```json
{
    "common": {
        "name": "adaptername",
        "version": "1.0.0",
        "platform": "Javascript/Node.js",
        "mode": "daemon",
        "tier": 3,
        "type": "misc-data",
        "connectionType": "cloud",
        "dataSource": "poll",
        "adminUI": { "config": "json" },
        "compact": true,
        "titleLang": { "en": "My Adapter", "de": "Mein Adapter" },
        "desc": { "en": "Description", "de": "Beschreibung" },
        "licenseInformation": { "license": "MIT", "type": "free" },
        "readme": "https://github.com/user/ioBroker.adapter#readme",
        "loglevel": "info",
        "keywords": ["keyword1", "keyword2"],
        "news": { "1.0.0": { "en": "Initial release", "de": "Erstveröffentlichung" } },
        "dependencies": [{ "js-controller": ">=6.0.11" }],
        "globalDependencies": [{ "admin": ">=7.6.20" }]
    }
}
```

### Key Fields Explained

| Field | Values | Description |
|-------|--------|-------------|
| `mode` | `daemon`, `schedule`, `once`, `none`, `extension` | How adapter is started. `subscribe` is deprecated. |
| `tier` | `1`, `2`, `3` | Adapter importance tier (required) |
| `connectionType` | `local`, `cloud`, `none` | How it connects to devices/services |
| `dataSource` | `poll`, `push`, `assumption`, `none` | How it receives data |
| `compact` | `true` / `false` | Compact mode support |
| `messagebox` | `true` / `false` | Supports `sendTo` messages |
| `blockly` | `true` / `false` | Blockly visual scripting support |
| `loglevel` | `silly`, `debug`, `info`, `warn`, `error` | Default log level |

### news Section

- Maximum **7** entries [repochecker warning].
- Every version listed must be published on npm.
- Translated into all supported languages.

### Native Config Defaults

- Every field used in jsonConfig.json **must** have a default in `native`.
- Keys in `encryptedNative`/`protectedNative` must also exist in `native`.

### Do NOT include these in io-package.json

- `$schema` (blacklisted)
- `common.singleton` (causes warning for non-wwwOnly)
- `common.wakeup` (deprecated)
- `common.automaticUpgrade` (disallowed)
- `common.schedule` with non-empty value for daemon adapters [W1114]
- `installedFrom` or `installedVersion` (artifacts from installed adapters)

---

## 22. README.md

- **Must be in English** [E6015] — German goes in `docs/de/README.md`.
- Must include: what the adapter does, device/manufacturer link, configuration instructions.
- Must include a changelog (or link to CHANGELOG.md).
- **Do not** include npm install instructions (`npm install iobroker.*`) [E6012].
- **Do not** include GitHub URL install instructions (`iobroker url ...`) [E6013].
- **Do not** include an `## Installation` section unless special setup is required [S6014].
- Copyright year must match the current or latest commit year.
- Multiple copyright lines must use trailing two spaces (not empty lines) [W6011].

### Changelog File Management [W6017–S6020]

| Situation | Rule |
|-----------|------|
| `CHANGELOG.md` exists in repo root | ❌ Move changelog into `README.md` [W6017] |
| Both `CHANGELOG.md` and `CHANGELOG_OLD.md` exist | ❌ Consolidate — only use README.md + CHANGELOG_OLD.md [W6018] |
| README changelog exceeds 20 entries, no `CHANGELOG_OLD.md` | Move older entries to `CHANGELOG_OLD.md` [W6019] |
| No `CHANGELOG_OLD.md` exists | Add `CHANGELOG_OLD.md` to store older entries [S6020] |

**Correct pattern**: Changelog entries live in `README.md`. Old entries are moved to `CHANGELOG_OLD.md`.

---

## 23. Package Files & npm Publishing

### package.json Requirements

- `name`: `iobroker.<adaptername>` (lowercase)
- `engines.node`: `">= 20"` (current recommendation)
- `main`: Points to compiled entry (e.g., `build/main.js` or `main.js`)
- **Do not** list `@types/*` packages in `dependencies` (only `devDependencies`) [W warning]
- **Do not** list `@iobroker/testing` in `dependencies` (only `devDependencies`) [Error]
- **Do not** list `admin` or `iobroker` as dependencies
- **Do not** use fixed version dependencies (e.g., `"1.2.3"`) — use ranges (`"^1.2.3"`)

### .npmignore or files field

- `.npmignore` **must** exist [E repochecker requirement].
- Or use `package.json` `"files"` field to whitelist published files.
- Ensure i18n directories are **not** excluded by `.npmignore` [E9506] and are covered by `"files"` [E9507].
- `package-lock.json` should exist in the repo.

### Dependency Hygiene

- Do **not** list `@types/*` packages in `dependencies` (only `devDependencies`) [W warning]
- Do **not** list `@iobroker/testing` in `dependencies` (only `devDependencies`) [Error]
- Do **not** list `admin` or `iobroker` as dependencies
- Do **not** use fixed version dependencies (e.g., `"1.2.3"`) — use ranges (`"^1.2.3"`) [S0047]
- **All packages imported via `require()` or `import` in source files must be listed in `dependencies`** — repochecker scans source files and warns for missing entries [W5042]
- Use the `node:` prefix for built-in Node.js modules: `node:fs`, `node:path`, `node:crypto` etc. [S5043]
- `@iobroker/plugin-sentry` is system-provided — do **not** add it to `dependencies`

### .releaseconfig.json

- If `.releaseconfig.json` exists, all listed plugins must be installed as devDependencies [E5036/W5037]

### .gitignore Requirements

- Must include: `node_modules/`, `build/` (for TS), `.dev-server/`, `iob_npm.done`
- Must **not** include `.commitinfo` file in repo [E905].

---

## 24. GitHub & Dependabot Configuration

### Required GitHub Setup

- Repository must have a default branch (`main` or `master`).
- GitHub Actions workflows for CI/CD (`test-and-release.yml`).

### Dependabot (recommended)

The repochecker suggests configuring dependabot:
- Ecosystems: `github-actions` and `npm` [S8901–S8914]
- Set `cooldown.default-days: 7` for npm to reduce supply chain risk [W8915]
- Set `open-pull-requests-limit` to at least 15 for `github-actions` [S8908]

> ⚠️ **Correct `cooldown` syntax**: The repochecker example `cooldown: { default: 7 }` is **wrong** and will fail GitHub's schema validator. The correct key is `default-days`:

```yaml
- package-ecosystem: "npm"
  directories: ["**/*"]
  schedule:
      interval: "daily"
  open-pull-requests-limit: 10
  cooldown:
      default-days: 7  # ✅ correct — NOT "default: 7"

- package-ecosystem: "github-actions"
  directory: "/"
  schedule:
      interval: "daily"
  open-pull-requests-limit: 15  # ✅ at least 15
```

### VS Code Settings (recommended)

`.vscode/settings.json` should have `json.schemas` for `io-package.json` and jsonConfig validation.

---

## 25. VIS Widgets (if applicable)

```
widgets/
├── adaptername.html
└── adaptername-widgetname/
    ├── js/adaptername-widgetname.js
    └── css/adaptername-widgetname.css
```

- **Remove the `widgets/` directory entirely** if not used (review requirement).
- Also remove unused `www/` and `docs/` directories.

---

## 26. Pre-Submission Checklist

Before submitting to the ioBroker repository:

1. ✅ Created with `npx @iobroker/create-adapter@latest`
2. ✅ `npx @iobroker/repochecker <repo-url>` passes with no errors
3. ✅ GitHub repo named `ioBroker.<adapterName>` (capital B)
4. ✅ npm package named `iobroker.<adaptername>` (lowercase)
5. ✅ English README.md with description + device/manufacturer link
6. ✅ License in `io-package.json`, README.md, AND `LICENSE` file
7. ✅ GitHub Actions CI/CD (`test-and-release.yml`) configured correctly
8. ✅ `type`, `tier`, `connectionType`, `dataSource`, `authors` set in io-package.json
9. ✅ State roles are specific (not generic `"state"`)
10. ✅ Published on npm + `iobroker` organization added as owner
11. ✅ `onUnload` cleans ALL resources
12. ✅ Compact mode tested (start → run → stop → restart)
13. ✅ `info.connection` implemented (if external connection exists)
14. ✅ Unused directories (`www`, `widgets`, `docs`) removed
15. ✅ Port attribute named `port`, IP attribute named `ip`
16. ✅ All object IDs sanitized (only `A-Za-z0-9-_.`)
17. ✅ All intermediate objects explicitly created in tree
18. ✅ Passwords/secrets in `encryptedNative` + `protectedNative`
19. ✅ Logs and README in English
20. ✅ i18n translations for all 11 languages, included in npm package
21. ✅ No example config keys (`option1`/`option2`) left in native or translations
22. ✅ Notifications (if any) translated into all languages
23. ✅ `js-controller >= 6.0.11` dependency declared
24. ✅ `admin >= 7.6.20` global dependency declared
25. ✅ Node.js `>= 20` in engines
26. ✅ `package-lock.json` committed to repo
27. ✅ All runtime imports listed in `dependencies` (not missing from package.json) [W5042]
28. ✅ Built-in Node.js modules use `node:` prefix (`node:fs`, `node:path`) [S5043]
29. ✅ No deprecated adapter methods (`createState`, `createChannel` etc.) [W5033]
30. ✅ Changelog in README.md (not CHANGELOG.md); older entries in CHANGELOG_OLD.md [W6017–S6020]
31. ✅ README has no `## Installation` section (unless special install needed) [S6014]
32. ✅ If using trusted publishing (`testing-action-deploy@v1`): `contents: write` + `id-token: write` permissions set, no `npm-token` [W3018/W3019]

---

## 27. Common AI Mistakes to Avoid

| ❌ Wrong | ✅ Correct |
|----------|-----------|
| `admin/index_m.html` (Materialize) | JSONConfig (`jsonConfig.json`) |
| `role: "state"` everywhere | Specific roles (`value.temperature`, `indicator.connected`) |
| States nested under states | `device → channel → state` hierarchy |
| Missing intermediate objects | Create ALL objects in tree explicitly |
| `setTimeout()` / `setInterval()` (native) | `this.setTimeout()` / `this.setInterval()` |
| `process.exit()` | `this.terminate()` |
| `console.log()` | `this.log.info()` / `this.log.debug()` |
| Plaintext passwords | `encryptedNative` + `protectedNative` |
| `setObject()` / `setObjectAsync()` | `setObjectNotExistsAsync()` / `extendObjectAsync()` |
| Ignoring `ack` flag | Check `!state.ack` in `onStateChange` |
| German README / log messages | English always |
| Special chars in object IDs | Sanitize: only `A-Za-z0-9-_` |
| Hardcoded i18n in jsonConfig | Use `i18n/` directory with JSON files |
| No standard tests | `@iobroker/testing` + `test-and-release.yml` |
| `node-cron` for simple polling | `this.setInterval()` with jitter |
| No unload cleanup | Clean ALL timers, connections, queues |
| Generating project scaffold | `npx @iobroker/create-adapter@latest` |
| Copying `io-package.json` from installed adapter | Contains `installedFrom` etc. |
| `admin/words.js` with jsonConfig | Remove — use i18n directory only |
| `common.schedule` on daemon adapter | Remove or leave empty |
| Missing `tier` field | Add `"tier": 3` (or appropriate level) |
| i18n not in npm package | Check `"files"` / `.npmignore` includes admin/i18n |
| Fixed version deps (`"1.2.3"`) | Use ranges (`"^1.2.3"`) |
| `createState()` / `createChannel()` / `createDevice()` | Use `setObjectNotExistsAsync()` instead [W5033] |
| `deleteState()` / `deleteChannel()` / `deleteDevice()` | Use `delObjectAsync()` instead [W5033] |
| `import fs from 'fs'` | Use `node:fs` prefix for built-ins [S5043] |
| Package used but not in `dependencies` | Add to `package.json` `dependencies` [W5042] |
| jsonConfig present but `adminUI.config` missing/wrong | Set `"adminUI": { "config": "json" }` [W5046] |
| `CHANGELOG.md` in repo root | Move entries to README.md; old ones to CHANGELOG_OLD.md |
| `npm-token` in deploy job with trusted publishing | Remove `npm-token` when using `testing-action-deploy@v1` [W3019] |

---

## 28. Reference Links

| Resource | URL |
|----------|-----|
| **AI Developer Guide** | https://github.com/Jey-Cee/iobroker-ai-developer-guide |
| **Adapter Creator** | `npx @iobroker/create-adapter@latest` |
| **Adapter Checker** | https://adapter-check.iobroker.in/ |
| **Repochecker CLI** | `npx @iobroker/repochecker <repo-url>` |
| **iobroker.dev** | https://www.iobroker.dev/ |
| **Repository Requirements** | https://github.com/ioBroker/ioBroker.repositories/blob/master/README.md |
| **Adapter Reference** | https://github.com/ioBroker/ioBroker.docs/blob/master/docs/en/dev/adapterref.md |
| **Object Schema** | https://github.com/ioBroker/ioBroker.docs/blob/master/docs/en/dev/objectsschema.md |
| **State Roles** | https://www.iobroker.net/docu/index-24.htm?page_id=5735&lang=en |
| **Repochecker Source** | https://github.com/ioBroker/ioBroker.repochecker |
| **Translator** | https://translator.iobroker.in/ |
| **Copilot Instructions** | https://github.com/DrozmotiX/ioBroker-Copilot-Instructions |
| **Forum (Tester)** | https://forum.iobroker.net/category/91/tester |
