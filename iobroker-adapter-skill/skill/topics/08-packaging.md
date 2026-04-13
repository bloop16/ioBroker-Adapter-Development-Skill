---
description: "ioBroker io-package.json complete reference, package.json requirements, npm publishing, GitHub setup, and Dependabot. Use when configuring adapter metadata, packaging, or CI/CD."
applyTo: "**/io-package.json,**/package.json,**/.github/**,**/.npmignore"
---

# ioBroker: Packaging, io-package.json & GitHub

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
        "titleLang": {
            "en": "My Adapter",
            "de": "Mein Adapter"
        },
        "desc": {
            "en": "What this adapter does",
            "de": "Was dieser Adapter macht"
        },
        "licenseInformation": {
            "license": "MIT",
            "type": "free"
        },
        "readme": "https://github.com/user/ioBroker.adaptername#readme",
        "loglevel": "info",
        "keywords": ["iobroker", "adaptername"],
        "news": {
            "1.0.0": {
                "en": "Initial release",
                "de": "Erstveröffentlichung"
            }
        },
        "dependencies": [{ "js-controller": ">=6.0.11" }],
        "globalDependencies": [{ "admin": ">=7.6.20" }]
    }
}
```

### Key Fields

| Field | Values | Description |
|-------|--------|-------------|
| `mode` | `daemon`, `schedule`, `once`, `none`, `extension` | How adapter is started. `subscribe` is deprecated. |
| `tier` | `1`, `2`, `3` | Adapter importance (1=core, 3=contributed). Required. |
| `connectionType` | `local`, `cloud`, `none` | Network path to device/service |
| `dataSource` | `poll`, `push`, `assumption`, `none` | How data is received |
| `compact` | `true` / `false` | Compact mode support |
| `messagebox` | `true` / `false` | Supports `sendTo` messages |
| `blockly` | `true` / `false` | Blockly visual scripting support |
| `loglevel` | `silly`, `debug`, `info`, `warn`, `error` | Default log level |

### Adapter Type Values (`common.type`)

| Type | Description |
|------|-------------|
| `alarm` | Security: home, car, boat |
| `climate-control` | Climate, heaters, air filters, water heaters |
| `communication` | RESTapi, websockets for other services |
| `date-and-time` | Schedules, calendars |
| `energy` | Energy metering |
| `metering` | Water, gas, oil metering |
| `garden` | Mowers, sprinklers |
| `general` | General purpose (admin, web, discovery) |
| `geoposition` | GPS/position tracking |
| `health` | Heart rate, blood pressure, body metrics |
| `hardware` | Arduino, ESP, Bluetooth multi-purpose hardware |
| `household` | Vacuum cleaners, kitchen appliances |
| `infrastructure` | Network, printers, phones, NAS |
| `iot-systems` | Comprehensive smart home systems |
| `lighting` | Light control |
| `logic` | Rules, scripts, parsers, scenes |
| `messaging` | Telegram, email, WhatsApp |
| `misc-data` | Contacts, system info, prices, currencies |
| `multimedia` | TV, AV receivers, multiroom audio, IR |
| `network` | Ping, network detection, UPnP |
| `protocols` | Communication protocols (MQTT, etc.) |
| `storage` | Logging, databases, file storage |
| `utility` | Backup, export/import tools |
| `vehicle` | Car integration |
| `visualization` | VIS, material, mobile |
| `visualization-icons` | Icon sets |
| `visualization-widgets` | VIS widgets |
| `weather` | Weather info, air quality |

### `connectionType` & `dataSource` Values

| `connectionType` | Meaning |
|-----------------|---------|
| `local` | Communication does not require cloud access |
| `cloud` | Communication goes via cloud service |
| `none` | No external connection |

| `dataSource` | Meaning |
|-------------|---------|
| `poll` | Status queried periodically (updates may be delayed) |
| `push` | ioBroker is notified when new data is available |
| `assumption` | Status cannot be determined; based on last command |

### `news` Section Rules

- Maximum **7** entries (repochecker warning if more).
- Every version listed must be published on npm.
- Translate into all supported languages.

### Native Config Defaults

- Every field used in `jsonConfig.json` **must** have a default value in `native`.
- Keys in `encryptedNative`/`protectedNative` must also exist in `native`.

### Fields NOT Allowed in `io-package.json`

- `$schema` — blacklisted
- `common.singleton` — causes warning for non-wwwOnly adapters
- `common.wakeup` — deprecated
- `common.automaticUpgrade` — disallowed
- `common.schedule` with a non-empty value for `daemon` adapters [W1114]
- `installedFrom` / `installedVersion` — artifacts from installed adapters, must not be committed

---

## 22. README.md Requirements

- **Must be in English** [E6015] — German goes in `docs/de/README.md`
- Must include: what the adapter does, device/manufacturer link, configuration instructions
- Must include changelog entries (see below)
- **Do not** include `npm install iobroker.*` instructions [E6012]
- **Do not** include `iobroker url ...` GitHub install instructions [E6013]
- **Do not** include an `## Installation` section unless special setup is required [S6014]
- Copyright year must match the current or latest commit year
- Multiple copyright lines must use trailing two spaces, not blank lines [W6011]

### Changelog File Management

| Situation | Rule |
|-----------|------|
| `CHANGELOG.md` in repo root | ❌ Move entries into `README.md` [W6017] |
| Both `CHANGELOG.md` and `CHANGELOG_OLD.md` exist | ❌ Consolidate — only README.md + CHANGELOG_OLD.md [W6018] |
| README changelog exceeds 20 entries, no `CHANGELOG_OLD.md` | Move older entries to `CHANGELOG_OLD.md` [W6019] |

**Correct pattern**: Recent changelog entries in `README.md`, older entries in `CHANGELOG_OLD.md`.

---

## 23. package.json Requirements

```json
{
    "name": "iobroker.adaptername",
    "version": "1.0.0",
    "description": "ioBroker adapter for ...",
    "author": { "name": "Your Name", "email": "your@email.com" },
    "main": "build/main.js",
    "engines": { "node": ">= 20" },
    "dependencies": {
        "axios": "^1.6.0"
    },
    "devDependencies": {
        "@iobroker/adapter-core": "^3.1.0",
        "@iobroker/testing": "^4.1.0",
        "@types/node": "^20.0.0"
    }
}
```

### Dependency Rules

| Rule | Detail |
|------|--------|
| No `@types/*` in `dependencies` | Only in `devDependencies` |
| No `@iobroker/testing` in `dependencies` | Only `devDependencies` [repochecker Error] |
| No `admin` or `iobroker` as dependencies | Never |
| No fixed versions (`"1.2.3"`) | Use ranges (`"^1.2.3"`) [S0047] |
| All `import`/`require` in source → in `dependencies` | Repochecker scans and warns [W5042] |
| Node.js built-ins use `node:` prefix | `node:fs`, `node:path`, `node:crypto` [S5043] |
| `@iobroker/plugin-sentry` NOT in `dependencies` | System-provided package |

### SemVer — Versioning Rules

ioBroker uses [Semantic Versioning](https://semver.org/):

| Part | Meaning |
|------|---------|
| **major** (x) | Breaking changes, incompatible with previous versions |
| **minor** (y) | New functionality, backwards compatible |
| **patch** (z) | Bug fixes, backwards compatible |

**Special rule for 0.y.z (pre-stable)**:  
Start at `0.1.0`. For any breaking change, increment `y`. For patches and minor additions, increment `z`.  
Once thoroughly tested in the `latest` repository, promote to `1.0.0` and request addition to `stable`.

### `.npmignore` Requirements

`.npmignore` **must** exist [repochecker requirement], OR use `"files"` in `package.json` to whitelist.

Ensure these are **included** in the published package:
- `admin/i18n/` [E9506, E9507]
- `admin/jsonConfig.json`
- `build/` (compiled output)

Typical `.npmignore`:
```
.github/
.vscode/
src/
test/
*.map
tsconfig*.json
```

### `.gitignore` Requirements

Must include:
```
node_modules/
build/
.dev-server/
iob_npm.done
```

Must **not** exclude `.commitinfo` [E905].

### `.releaseconfig.json`

If present, all listed plugins must be installed as `devDependencies` [E5036/W5037].

---

## 24. GitHub & Dependabot Configuration

### Required GitHub Setup

- Repository default branch: `main` or `master`
- GitHub Actions: `test-and-release.yml` from Adapter Creator

### Dependabot (`dependabot.yml`)

```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directories: ["**/*"]
    schedule:
        interval: "daily"
    open-pull-requests-limit: 10
    cooldown:
        default-days: 7   # ✅ Correct key — NOT "default: 7"

  - package-ecosystem: "github-actions"
    directory: "/"
    schedule:
        interval: "daily"
    open-pull-requests-limit: 15  # Must be at least 15 [S8908]
```

> ⚠️ The repochecker example uses `cooldown: { default: 7 }` which is **wrong** — GitHub's schema validator requires `default-days`.

### VS Code Settings (recommended)

`.vscode/settings.json` with JSON schema validation:
```json
{
    "json.schemas": [
        {
            "fileMatch": ["io-package.json"],
            "url": "https://raw.githubusercontent.com/ioBroker/ioBroker.js-controller/master/schemas/io-package.json"
        }
    ]
}
```

---

## 25. VIS Widgets (if applicable)

```
widgets/
├── adaptername.html
└── adaptername-widgetname/
    ├── js/adaptername-widgetname.js
    └── css/adaptername-widgetname.css
```

**If not used**: Remove the `widgets/` directory entirely (repository review requirement).  
Also remove unused `www/` and `docs/` directories.

---

## Related Sub-Skills

- [01-setup.md](01-setup.md) — Naming conventions
- [04-admin-ui.md](04-admin-ui.md) — JSONConfig and i18n packaging
- [05-security.md](05-security.md) — encryptedNative in io-package.json
- [09-submission.md](09-submission.md) — Repository submission workflow
