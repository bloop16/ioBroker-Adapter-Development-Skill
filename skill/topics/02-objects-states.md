---
description: "ioBroker object hierarchy, state definitions, state roles, and the ack flag. Use when creating or reviewing adapter objects and states."
applyTo: "**/*.ts,**/*.js"
---

# ioBroker: Objects, States, Roles & the `ack` Flag

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

### Deprecated Methods — Do Not Use [W5033]

| ❌ Deprecated | ✅ Use Instead |
|--------------|--------------|
| `createState()` | `setObjectNotExistsAsync()` |
| `createChannel()` | `setObjectNotExistsAsync()` |
| `createDevice()` | `setObjectNotExistsAsync()` |
| `deleteState()` | `delObjectAsync()` |
| `deleteChannel()` | `delObjectAsync()` |
| `deleteDevice()` | `delObjectAsync()` |

---

## 4. State Definitions — Mandatory Fields

Every state **must** have these `common` attributes:

```typescript
{
    type: "state",
    common: {
        name: "Human readable name",     // REQUIRED — string or { en: "...", de: "..." }
        type: "number",                   // REQUIRED: "string"|"number"|"boolean"|"array"|"object"|"json"|"mixed"|"file"
        role: "value.temperature",        // REQUIRED: must be specific (never "state")
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

### Type Notes

- `"array"`, `"object"`, `"mixed"`, and `"file"` values must be serialized via `JSON.stringify()`.
- Use `role: "json"` with `type: "string"` for structured JSON data stored as a string.
- Use `common.states` for finite enum sets:
  ```typescript
  common: { type: "number", role: "value", states: { 0: "Offline", 1: "Online", 2: "Busy" } }
  ```

---

## 5. State Roles — NEVER Use Generic `"state"`

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

### Role Assignment Rules

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

### Handling in `onStateChange`

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

### ack Rules

- **All values FROM the device/API** → always `ack: true`
- **User commands TO the device** → received with `ack: false`, confirmed with `ack: true` after execution
- **Never ignore the ack flag** — always check `!state.ack` in `onStateChange`
- Some protocol bridges may process both ack states; this requires explicit configuration

---

## Related Sub-Skills

- [01-setup.md](01-setup.md) — Naming conventions and project setup
- [03-lifecycle.md](03-lifecycle.md) — Adapter lifecycle and timer management
- [04-admin-ui.md](04-admin-ui.md) — Admin UI and translations
