---
description: "ioBroker adapter patterns: scheduling & polling, message handling (sendTo), compact mode, and Device Manager integration. Use when implementing polling loops, inter-adapter communication, or compact mode support."
applyTo: "**/*.ts,**/*.js,**/io-package.json"
---

# ioBroker: Scheduling, Messaging & Compact Mode

## 12. Scheduling & Polling

### Simple Intervals — Use Adapter Timers

```typescript
// Start polling after adapter is ready
this.pollingInterval = this.setInterval(() => this.fetchData(), this.config.pollingInterval * 1000);
```

### Anti-Thundering-Herd: Add Jitter

When many users run the same adapter, all instances polling at exactly the same second can DDoS external APIs. Add random jitter to stagger startup:

```typescript
const baseMs = this.config.pollingInterval * 1000;
const jitter = Math.floor(Math.random() * 5000); // Up to 5s random delay
this.setTimeout(() => {
    this.pollingInterval = this.setInterval(() => this.fetchData(), baseMs);
}, jitter);
```

### Avoid Scheduling Libraries for Simple Intervals

- **Do not** use `node-cron`, `node-schedule`, or similar for basic periodic polling.
- `this.setInterval()` is sufficient and has the side effect of staggering requests across instances.
- **Exception**: Complex cron patterns (astronomical events, business hours) may justify a scheduling library.

### Request Throttling (for external APIs)

Prevent concurrent request pile-up when the previous request hasn't finished:

```typescript
private activeRequests = 0;
private readonly maxConcurrent = 3;
private queue: Array<{ resolve: () => void; reject: (err: Error) => void }> = [];

private async throttled<T>(fn: () => Promise<T>): Promise<T> {
    if (this.activeRequests >= this.maxConcurrent) {
        await new Promise<void>((resolve, reject) => this.queue.push({ resolve, reject }));
    }
    this.activeRequests++;
    try {
        return await fn();
    } finally {
        this.activeRequests--;
        this.queue.shift()?.resolve();
    }
}

// Usage
private async fetchData(): Promise<void> {
    if (this.isShuttingDown) return;
    await this.throttled(async () => {
        const data = await this.apiClient.getData();
        await this.processData(data);
    });
}
```

---

## 13. Message Handling (`messagebox` / `sendTo`)

For adapters that receive commands from other adapters, scripts, or the Admin UI.

### Enable in `io-package.json`

```json
{
    "common": {
        "messagebox": true
    }
}
```

### Handle Messages in Adapter Code

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
                await this.testConnection(obj.message as { host: string; port: number });
                this.sendTo(obj.from, obj.command, { success: true }, obj.callback);
            } catch (err: any) {
                this.sendTo(obj.from, obj.command, { success: false, error: err.message }, obj.callback);
            }
            break;
        case "getDevices":
            this.sendTo(obj.from, obj.command, { devices: this.deviceList }, obj.callback);
            break;
        default:
            this.log.warn(`Unknown command: ${obj.command}`);
    }
}
```

### Calling from Admin UI (`jsonConfig.json`)

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

All adapters should support compact mode to allow multiple instances to run in a single process.

### Enable in `io-package.json`

```json
{
    "common": {
        "compact": true
    }
}
```

### Entry Point Pattern

```typescript
// TypeScript
if (require.main !== module) {
    // Compact mode: export factory function
    module.exports = (options: Partial<utils.AdapterOptions> | undefined) => new MyAdapter(options);
} else {
    // Standalone: create instance directly
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

- ✅ `this.setTimeout()` / `this.setInterval()` (never native Node.js timers)
- ✅ `this.terminate()` (never `process.exit()`)
- ✅ Clean up ALL resources in `onUnload` (see [03-lifecycle.md](03-lifecycle.md))
- ✅ No global mutable state shared between instances
- ✅ Entry point exports factory function (not just creates instance)
- ✅ Start → Run → Stop → Restart cycle works without side effects

---

## 17. Device Manager Integration (optional)

For adapters managing multiple physical devices with a standardized device management UI.

### Enable in `io-package.json`

```json
{
    "common": {
        "supportedMessages": {
            "deviceManager": true
        }
    },
    "globalDependencies": [{ "admin": ">=7.8.19" }]
}
```

Use the `@iobroker/dm-utils` library for the standardized device management adapter API.

---

## Related Sub-Skills

- [03-lifecycle.md](03-lifecycle.md) — Timer management and onUnload
- [04-admin-ui.md](04-admin-ui.md) — sendTo button in JSONConfig
- [07-tooling.md](07-tooling.md) — Error handling patterns
