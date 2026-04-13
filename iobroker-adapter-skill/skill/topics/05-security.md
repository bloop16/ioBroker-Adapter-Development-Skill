---
description: "ioBroker adapter security: password encryption, secret management, and encryptedNative. Use when handling passwords, API keys, tokens, or any sensitive configuration."
applyTo: "**/io-package.json,**/*.ts,**/*.js"
---

# ioBroker: Passwords & Secrets

## 11. Passwords & Secrets

**Never** store passwords, API keys, or tokens in plaintext in `native` without protection.

### Configuration in `io-package.json`

```json
{
    "encryptedNative": ["password", "apiKey", "token"],
    "protectedNative": ["password", "apiKey", "token"],
    "native": {
        "password": "",
        "apiKey": "",
        "token": ""
    }
}
```

### Purpose of Each Field

| Field | Purpose |
|-------|---------|
| `encryptedNative` | Fields are encrypted at rest in the ioBroker database |
| `protectedNative` | Fields are hidden from API responses and Admin UI network traffic |

- Always list sensitive fields in **both** `encryptedNative` AND `protectedNative`.
- Every key in `encryptedNative`/`protectedNative` must also exist in `native` (with an empty default).
- The adapter receives the **decrypted** value at runtime via `this.config.*` — no manual decryption needed.

### Required Dependencies

For `encryptedNative` to work, declare minimum versions in `io-package.json`:

```json
{
    "common": {
        "dependencies": [{ "js-controller": ">=3.0.0" }],
        "globalDependencies": [{ "admin": ">=4.0.9" }]
    }
}
```

> Note: The current minimum is `js-controller >= 6.0.11` and `admin >= 7.6.20`. Check [08-packaging.md](08-packaging.md) for current required versions.

### Admin UI — Password Field

In `jsonConfig.json`, use `type: "password"` for secret fields — this auto-masks input:

```json
{
    "apiKey": {
        "type": "password",
        "label": "API Key",
        "sm": 12,
        "md": 6
    }
}
```

### Repochecker Warning

The repochecker scans `native` for keys containing: `pass`, `key`, `secret`, `token`, `auth`, `credential`.  
If any such key is **not** listed in `encryptedNative` + `protectedNative`, a warning is issued.

### Accessing Secrets in Adapter Code

```typescript
// Decrypted values are available directly — no manual decryption needed
const apiKey = this.config.apiKey;   // Already decrypted at runtime
const password = this.config.password;

// Use them for API calls
const auth = Buffer.from(`${this.config.username}:${password}`).toString("base64");
```

---

## Related Sub-Skills

- [04-admin-ui.md](04-admin-ui.md) — Password field in JSONConfig UI
- [08-packaging.md](08-packaging.md) — io-package.json complete reference
