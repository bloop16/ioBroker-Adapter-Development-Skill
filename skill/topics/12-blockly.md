---
description: "ioBroker Blockly block integration for adapters: required flags, admin/blockly.js structure, generator patterns, and validation checklist."
applyTo: "**/admin/blockly.js,**/io-package.json,**/package.json"
---

# ioBroker: Blockly Blocks for Adapters

Use this topic when an adapter exposes custom Blockly blocks (usually via `sendTo`).

Primary references:
- https://developers.google.com/blockly
- https://developers.google.com/blockly/guides/create-custom-blocks/generating-code
- https://github.com/ioBroker/ioBroker.repositories/blob/master/REVIEW_CHECKLIST.md
- https://github.com/ioBroker/ioBroker.repochecker

Example implementation:
- https://github.com/ticaki/ioBroker.icloud/blob/main/AGENT_BLOCKLY.md

If there is a conflict, follow current repochecker/review-checklist behavior.

## 1) Adapter Prerequisites

`io-package.json` in `common` must contain:

```json
{
    "blockly": true,
    "messagebox": true
}
```

- `blockly: true` enables loading of `admin/blockly.js` by JavaScript adapter Blockly integration.
- `messagebox: true` is required for `sendTo` command handling.

## 2) Packaging Prerequisites

- Ensure `admin/blockly.js` is included in the published npm package.
- If using `"files"` in `package.json`, verify the glob already covers admin JavaScript files.
- Do not assume; confirm with package contents/check scripts.

## 3) Required File Structure

Path:

```text
admin/blockly.js
```

Typical block section order:
1. goog preamble + translate helper
2. optional helper functions (instance dropdown etc.)
3. per block:
   - `Blockly.Words[...]`
   - `Blockly.Sendto.blocks[...]`
   - `Blockly.Blocks[...]`
   - `Blockly.JavaScript[...]`

## 4) Language and Translation Rules

- Keep Blockly labels/help texts in `Blockly.Words`.
- Provide all 11 ioBroker languages where applicable:
  - `en`, `de`, `ru`, `pt`, `nl`, `fr`, `it`, `es`, `pl`, `uk`, `zh-cn`

## 5) Generator and Block-Type Rules

- Statement blocks typically return generated code string.
- Value blocks typically return `[code, ORDER]` tuple.
- Keep generated code resilient for optional values (avoid malformed object literals).
- Use adapter instance selection safely (dropdown helper/fallback options).
- Verify generator output against current Blockly docs and working adapter runtime behavior.

## 6) Browser-Context Compatibility

`admin/blockly.js` runs in browser/blockly context.

- Prefer conservative JavaScript syntax compatible with project tooling.
- Avoid Node-specific runtime assumptions.
- Keep file focused and dependency-free.

## 7) Validation Checklist (Before PR)

1. `io-package.json` contains `blockly: true` and `messagebox: true`.
2. `admin/blockly.js` exists and is included in npm package.
3. Block appears in Blockly toolbox.
4. Generated code is valid for both default and optional fields.
5. Build/lint/tests are green.
6. Runtime behavior validated in dev-server before release steps.

## Related Sub-Skills

- [04-admin-ui.md](04-admin-ui.md) — JSONConfig and translation handling
- [08-packaging.md](08-packaging.md) — package include rules
- [10-dev-server.md](10-dev-server.md) — runtime live validation
- [11-pr-checklist.md](11-pr-checklist.md) — PR quality gate
