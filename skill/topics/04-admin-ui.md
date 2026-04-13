---
description: "ioBroker Admin UI using JSONConfig and i18n translations. Use when creating or modifying the adapter configuration UI."
applyTo: "**/jsonConfig.json,**/jsonConfig.json5,**/io-package.json,**/i18n/**"
---

# ioBroker: Admin UI (JSONConfig) & Translations

## 9. Admin UI — JSONConfig

### Always Use JSONConfig

- **Never** generate `admin/index_m.html` or `admin/index.html` (deprecated Admin2/Materialize).
- Use `admin/jsonConfig.json` or `admin/jsonConfig.json5`.
- Set in `io-package.json`: `"adminUI": { "config": "json" }` [W5046]
- Remove obsolete files `admin/index.html`, `admin/index_m.html`, `admin/style.css` [W5047]
- Do **not** use `admin/words.js` — it is considered outdated when jsonConfig is used [W5039]

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

### Tab-Based Structure (for complex configurations)

```json
{
    "type": "tabs",
    "i18n": true,
    "items": {
        "connectionTab": {
            "type": "panel",
            "label": "Connection",
            "items": {
                "host": { "type": "text", "label": "IP Address", "sm": 12, "md": 6 },
                "port": { "type": "number", "label": "Port", "sm": 12, "md": 6, "min": 1, "max": 65535 }
            }
        },
        "advancedTab": {
            "type": "panel",
            "label": "Advanced",
            "items": {
                "pollingInterval": { "type": "number", "label": "Polling Interval (s)", "sm": 12 }
            }
        }
    }
}
```

### Common Field Types

| Type | Use for |
|------|---------|
| `text` | Host, username, API path |
| `number` | Port, interval, timeout (with `min`/`max`) |
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
        "sm": 12,
        "md": 6,
        "lg": 4
    }
}
```

### sendTo Button (Admin UI → Adapter)

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

### Repochecker-Enforced Rules

- Port attribute **must** be named `port` [enforced]
- IP attribute **must** be named `ip` [enforced]
- If `admin/jsonConfig.json` exists, `common.adminUI.config` **must** be `"json"` [W5046]
- If `native` has config properties but `adminUI.config` is `"none"`, the config will be unusable [W1113]
- jsonConfig is validated against the official JSON schema automatically [E5512]
- Do **not** use deprecated methods `createState`, `createChannel`, `createDevice` [W5033]

### io-package.json entry

```json
{
    "common": {
        "adminUI": { "config": "json" }
    }
}
```

---

## 10. Translations (i18n)

### Required Directory Structure

```
admin/
├── jsonConfig.json
└── i18n/
    ├── en.json          ← MANDATORY
    ├── de.json          ← Required by repochecker
    ├── ru.json
    ├── pt.json
    ├── nl.json
    ├── fr.json
    ├── it.json
    ├── es.json
    ├── pl.json
    ├── uk.json
    └── zh-cn.json
```

### File Format (flat key-value)

```json
{
    "IP Address": "IP Address",
    "Port": "Port",
    "Polling Interval (s)": "Polling Interval (s)",
    "Test Connection": "Test Connection"
}
```

### Rules

- Set `"i18n": true` in `jsonConfig.json` (at root level).
- **English (`en.json`) is mandatory**; `de.json` is required by repochecker.
- All other 9 languages are strongly recommended.
- **Do not** hardcode language strings directly in `jsonConfig.json` — use key references.
- Do not leave example keys (`option1`, `option2`) in translation files [W5041].
- i18n directories **must be included** in the npm package [E9506, E9507]:
  - Check `"files"` in `package.json` includes `admin/i18n/`
  - Check `.npmignore` does NOT exclude `admin/i18n/`

### Translation Tool

Use https://translator.iobroker.in/ to auto-translate all strings from English into all required languages.

### Notifications Translations

Notification texts in `io-package.json` must also be translated into **all** 11 supported languages [E1112]:
```json
"notifications": [{
    "categories": [{
        "name": {
            "en": "Updates available",
            "de": "Updates verfügbar",
            "ru": "Доступны обновления",
            "pt": "Atualizações disponíveis",
            "nl": "Updates beschikbaar",
            "fr": "Mises à jour disponibles",
            "it": "Aggiornamenti disponibili",
            "es": "Actualizaciones disponibles",
            "pl": "Dostępne aktualizacje",
            "uk": "Доступні оновлення",
            "zh-cn": "有可用更新"
        }
    }]
}]
```

---

## Related Sub-Skills

- [05-security.md](05-security.md) — Password fields and encryption
- [06-patterns.md](06-patterns.md) — Message handling for sendTo buttons
- [08-packaging.md](08-packaging.md) — io-package.json, npm package inclusion rules
