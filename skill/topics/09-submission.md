---
description: "ioBroker adapter pre-submission checklist, common AI mistakes, repository submission workflow (latest/stable), and reference links. Use before publishing or submitting an adapter."
applyTo: "**/io-package.json,**/package.json,**/README.md"
---

# ioBroker: Submission, Checklist & Repository Workflow

## 26. Pre-Submission Checklist

Run `npx @iobroker/repochecker <repo-url>` and fix **all** errors before submitting a PR.

### Project Structure
1. ✅ Created with `npx @iobroker/create-adapter@latest`
2. ✅ `npx @iobroker/repochecker <repo-url>` passes with no errors
3. ✅ GitHub repo named `ioBroker.<adapterName>` (capital B)
4. ✅ npm package named `iobroker.<adaptername>` (lowercase)
5. ✅ Unused directories (`www`, `widgets`, `docs`) removed

### Metadata
6. ✅ English README.md with description + device/manufacturer link
7. ✅ License in `io-package.json`, README.md, AND `LICENSE` file
8. ✅ `type`, `tier`, `connectionType`, `dataSource`, `authors` set in `io-package.json`
9. ✅ `js-controller >= 6.0.11` dependency declared
10. ✅ `admin >= 7.6.20` global dependency declared
11. ✅ Node.js `>= 20` in `engines`
12. ✅ Changelog in README.md (not `CHANGELOG.md`); older entries in `CHANGELOG_OLD.md`
13. ✅ README has no `## Installation` section (unless special install needed) [S6014]

### Code Quality
14. ✅ State roles are specific — not generic `"state"`
15. ✅ All object IDs sanitized (only `A-Za-z0-9-_.`)
16. ✅ All intermediate objects explicitly created in the tree
17. ✅ `onUnload` cleans ALL resources (timers, connections, queues)
18. ✅ Compact mode tested: start → run → stop → restart cycle works
19. ✅ `info.connection` implemented (if external connection exists)
20. ✅ Port attribute named `port`, IP attribute named `ip`

### Security
21. ✅ Passwords/secrets in `encryptedNative` + `protectedNative`
22. ✅ No deprecated methods (`createState`, `createChannel` etc.) used [W5033]

### Translations & i18n
23. ✅ Logs and README in English
24. ✅ i18n translations for all 11 languages (en, de, ru, pt, nl, fr, it, es, pl, uk, zh-cn)
25. ✅ i18n directories included in the npm package (`"files"` / `.npmignore`)
26. ✅ No example config keys (`option1`/`option2`) left in native or translations [W5041]
27. ✅ Notifications (if any) translated into all 11 languages [E1112]

### Dependencies & CI
28. ✅ Published on npm + `iobroker` organisation added as owner (see below)
29. ✅ `package-lock.json` committed to repo
30. ✅ All runtime imports listed in `dependencies` [W5042]
31. ✅ Built-in Node.js modules use `node:` prefix (`node:fs`, `node:path`) [S5043]
32. ✅ GitHub Actions CI/CD (`test-and-release.yml`) configured correctly
33. ✅ If using trusted publishing: `contents: write` + `id-token: write` set, no `npm-token` [W3018/W3019]

---

## Repository Submission Workflow

### Step 1: Publish to npm

```bash
npm publish
```

You must create an npm account first if you don't have one.

### Step 2: Add `iobroker` Organisation as npm Owner

This is **required** by the ioBroker repository. The organisation is granted publish rights only for emergencies:

```bash
npm owner add bluefox iobroker.adaptername
```

> Wait 1–2 days for `bluefox` to accept the invite. If the invite expires, send it again.

### Step 3: Add to the Latest Repository

**Option A — CLI:**
```bash
# Fork https://github.com/ioBroker/ioBroker.repositories and clone it
npm i
npm run addToLatest -- --name adaptername --type misc-data
# Commit changes to sources-dist.json and open a PR
```

**Option B — Web frontend:**
1. Go to https://www.iobroker.dev/
2. Log in with GitHub
3. Open your adapter
4. Click **Manage** → **ADD TO LATEST**

### Step 4: Requirements for Latest Repository

1. GitHub repository named `ioBroker.<adaptername>` with topics set
2. No "ioBroker" or "Adapter" in the repository title
3. English README.md with manufacturer/device link
4. Predefined license
5. All unused directories removed (`www`, `widgets`, `docs`)
6. GitHub Actions tests passing (at least package + integration tests)
7. Valid `type` in `io-package.json`
8. Valid `connectionType` and `dataSource` if applicable
9. All states have valid specific roles (not generic `"state"`)
10. `iobroker` organisation added as npm owner
11. Adapter available on npmjs.com

### Step 5: Forum Thread (for Latest)

Post a thread in https://forum.iobroker.net/ announcing your adapter and asking users to test it. Reference the GitHub URL for installation via Admin UI (not npm command or GitHub URL).

### Step 6: Add to Stable Repository

After testing by multiple users in different environments:

```bash
# Fork https://github.com/ioBroker/ioBroker.repositories and clone it
npm i
npm run addToStable -- --name adaptername --version 1.0.0
# Commit changes to sources-dist-stable.json and open a PR
```

**Requirements for Stable** (in addition to Latest requirements):
1. Adapter was in the Latest repository first
2. Forum thread asking for testers (https://forum.iobroker.net/category/91/tester)
3. Positive feedback from multiple users in different environments
4. If device can be automatically discovered (USB, IP), implement or request Discovery adapter integration

---

## 27. Common AI Mistakes to Avoid

| ❌ Wrong | ✅ Correct |
|----------|-----------|
| `admin/index_m.html` (Materialize) | `admin/jsonConfig.json` with `adminUI.config: "json"` |
| `role: "state"` everywhere | Specific roles (`value.temperature`, `indicator.connected`) |
| States nested under states | `device → channel → state` hierarchy |
| Missing intermediate objects | Create ALL objects in tree explicitly |
| `setTimeout()` / `setInterval()` (native) | `this.setTimeout()` / `this.setInterval()` |
| `process.exit()` | `this.terminate()` |
| `console.log()` | `this.log.info()` / `this.log.debug()` |
| Plaintext passwords in native | `encryptedNative` + `protectedNative` |
| `setObject()` / `setObjectAsync()` | `setObjectNotExistsAsync()` / `extendObjectAsync()` |
| Ignoring `ack` flag | Check `!state.ack` in `onStateChange` |
| German README or log messages | English always |
| Special chars in object IDs | Sanitize: only `A-Za-z0-9-_.` |
| Hardcoded strings in jsonConfig | Use `i18n/` directory with JSON key files |
| No standard tests | `@iobroker/testing` + `test-and-release.yml` |
| `node-cron` for simple polling | `this.setInterval()` with jitter |
| No cleanup in `onUnload` | Clean ALL timers, connections, queues |
| Generating project scaffold | `npx @iobroker/create-adapter@latest` |
| Copying `io-package.json` from installed adapter | Contains `installedFrom` — do not copy |
| `admin/words.js` with jsonConfig | Remove — use `admin/i18n/` only |
| `common.schedule` set on daemon adapter | Remove or leave empty |
| Missing `tier` field | Add `"tier": 3` (or appropriate level) |
| i18n not in npm package | Check `"files"` / `.npmignore` includes `admin/i18n` |
| Fixed version deps (`"1.2.3"`) | Use ranges (`"^1.2.3"`) |
| `createState()` / `createChannel()` / `createDevice()` | `setObjectNotExistsAsync()` [W5033] |
| `deleteState()` / `deleteChannel()` / `deleteDevice()` | `delObjectAsync()` [W5033] |
| `import fs from 'fs'` | `import * as fs from 'node:fs'` [S5043] |
| Package used but not in `dependencies` | Add to `package.json` `dependencies` [W5042] |
| jsonConfig present but `adminUI.config` missing | Set `"adminUI": { "config": "json" }` [W5046] |
| `CHANGELOG.md` in repo root | Move entries to README.md + CHANGELOG_OLD.md |
| `npm-token` with trusted publishing | Remove `npm-token` for `testing-action-deploy@v1` [W3019] |

---

## 28. Reference Links

| Resource | URL |
|----------|-----|
| **AI Developer Guide** | https://github.com/Jey-Cee/iobroker-ai-developer-guide |
| **Adapter Creator** | `npx @iobroker/create-adapter@latest` |
| **Adapter Checker (web)** | https://adapter-check.iobroker.in/ |
| **Repochecker CLI** | `npx @iobroker/repochecker <repo-url>` |
| **iobroker.dev** | https://www.iobroker.dev/ |
| **Repository Requirements** | https://github.com/ioBroker/ioBroker.repositories/blob/master/README.md |
| **ioBroker.repositories** | https://github.com/ioBroker/ioBroker.repositories/tree/master |
| **Adapter Developer Guide** | https://github.com/ioBroker/ioBroker.docs/blob/master/docs/en/dev/adapterdev.md |
| **Adapter Reference** | https://github.com/ioBroker/ioBroker.docs/blob/master/docs/en/dev/adapterref.md |
| **Object Schema** | https://github.com/ioBroker/ioBroker.docs/blob/master/docs/en/dev/objectsschema.md |
| **State Roles** | https://github.com/ioBroker/ioBroker/blob/master/doc/STATE_ROLES.md |
| **Repochecker Source** | https://github.com/ioBroker/ioBroker.repochecker |
| **Translator** | https://translator.iobroker.in/ |
| **Copilot Instructions** | https://github.com/DrozmotiX/ioBroker-Copilot-Instructions |
| **Forum (Tester)** | https://forum.iobroker.net/category/91/tester |
| **ioBroker RAG** | https://github.com/Skeletor-ai/iobroker-rag |

---

## Related Sub-Skills

- [08-packaging.md](08-packaging.md) — io-package.json, package.json, npm publishing
- [01-setup.md](01-setup.md) — Naming conventions
- [02-objects-states.md](02-objects-states.md) — Object hierarchy and state roles
