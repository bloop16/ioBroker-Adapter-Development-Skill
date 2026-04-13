---
description: "ioBroker adapter lifecycle management: timers, onUnload cleanup, connection state, and running modes. Use when writing lifecycle code or debugging resource leaks."
applyTo: "**/*.ts,**/*.js,**/io-package.json"
---

# ioBroker: Adapter Lifecycle, Timers & Connection State

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

### Graceful Shutdown Pattern (adapters with active requests)

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

### Lifecycle Rules

- **Every** timer, interval, WebSocket, HTTP client, TCP server, and file handle must be cleaned up in `onUnload`.
- The `callback()` **must always be called**, even on errors — use `finally` or a force timeout.
- Use an `isShuttingDown` flag to prevent new operations after unload begins.
- Test with **compact mode**: Start → Run → Stop → Restart must all work cleanly.

---

## 8. Connection State (`info.connection`)

**Required** for all adapters that connect to external services, devices, or protocols.

### Declaration in `io-package.json`

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

Set to `false` on adapter start and on unload. Set to `true` after a successful connection is established.

### For Multi-Device Adapters

```typescript
this.onlineDevices[deviceId] = isAlive;
const anyOnline = Object.values(this.onlineDevices).some(v => v);
await this.setStateAsync("info.connection", anyOnline, true);
```

The `info.connection` state controls the Admin UI indicator:
- `true` → green (connected)
- `false` + adapter running → yellow (running but disconnected)
- adapter stopped → red

---

## Running Modes Reference

Defined via `common.mode` in `io-package.json`:

| Mode | Behavior | Use Case |
|------|----------|----------|
| `daemon` | Always running, restarted on exit | Most adapters — continuous data/connection |
| `schedule` | Started by cron schedule in `common.schedule` | Periodic data collection (e.g. weather) |
| `once` | Started every time the instance object changes, not restarted | One-shot tasks |
| `none` | Not started automatically | Libraries, helper adapters |
| `extension` | Loaded as extension of another adapter | Special use |
| `subscribe` | ⚠️ **Deprecated** — do not use | — |

### Important Notes

- `daemon` adapters must **not** have a non-empty `common.schedule` [W1114].
- `schedule` adapters should define a valid cron expression in `common.schedule` (e.g. `"0 * * * *"` for hourly).
- If you change `mode` after publishing, users must delete and re-create all instances.

### Adapter Start Flags (for debugging)

When starting the adapter manually via `node main.js`:

| Flag | Effect |
|------|--------|
| `--install` | Starts adapter even if no configuration exists (used for install scripts) |
| `--force` | Starts adapter even if disabled in configuration |
| `--logs` | Shows logs in console (they normally only appear in the log table) |

Example: `node main.js 0 debug --force`  
(Instance 0, log level debug, force start)

---

## Related Sub-Skills

- [02-objects-states.md](02-objects-states.md) — Object hierarchy and state management
- [06-patterns.md](06-patterns.md) — Polling patterns, compact mode, scheduling
- [07-tooling.md](07-tooling.md) — Error handling and logging
