---
description: "ioBroker pull-request checklist for community adapters: what must not change in PRs, required checks, fork-specific repochecker findings, and release responsibility."
applyTo: "**/README.md,**/io-package.json,**/package.json,**/.github/workflows/**"
---

# ioBroker: Pull Request Checklist (Community Adapters)

Use this checklist before opening or updating a PR for an ioBroker adapter.

Primary sources of truth:
- https://github.com/ioBroker/ioBroker.repositories/blob/master/REVIEW_CHECKLIST.md
- https://github.com/ioBroker/ioBroker.repochecker

If this checklist conflicts with current repochecker output, follow the official sources above.

## 1) Do Not Change These In Normal PRs

- Do not bump `version` in `package.json`.
- Do not bump `version` in `io-package.json`.
- Do not add new `news` entries in `io-package.json`.
- Do not add upgrade messages intended for release-only announcements.

Reason: these are typically handled by release scripts during the maintainer release flow.

## 2) README Changelog Format In PRs

Keep the active changelog section directly under:

```markdown
### **WORK IN PROGRESS**
- (Username) Description of change
```

Rules:
- No new version header for an unreleased PR entry.
- Every functional change in the PR must be listed.
- Keep format `- (Username) ...` consistently.

## 3) Mandatory Pre-PR Checks

Run and fix failures before PR updates:

```bash
npm run build
npm run test
npm run lint
```

Additionally run repochecker:

```bash
npx @iobroker/repochecker <repo-url> <branch>
```

Online alternative: https://adapter-check.iobroker.in/

## 4) Code-Level PR Quality Rules

- No `console.log`; use adapter logger only.
- Use `this.terminate()` instead of `process.exit()`.
- Ensure `onUnload` cleans all timers, intervals, sockets, and clients.
- Use robust error text extraction, for example:
  - `error instanceof Error ? error.message : String(error)`
- Avoid synchronous filesystem operations in hot paths.
- Ensure sensitive fields are in both `encryptedNative` and `protectedNative`.

## 5) Git and PR Workflow Rules

- Never commit/push directly to `main`; use feature branches.
- Do not create PRs automatically unless explicitly requested by the user.
- Use descriptive branch names (`fix/...`, `feature/...`).

## 6) Fork PRs: Expected Repochecker Findings

For fork-based PRs, some checks can appear although upstream is correct. Re-verify after merge to upstream.

Typical examples from community practice:
- `E0019` repository URL points to fork
- `E4005` / `E4007` icon/meta URL in fork context
- `E3007` workflow push trigger on fork default branch
- `E8002` missing topics on fork
- `W2002` local version mismatch vs npm during PR phase

Treat these as context-dependent, not automatic ignores. Validate against the current official checklist and target repository state.

## 7) Documentation Expectations

- Keep main `README.md` concise; place detailed docs in `docs/`.
- `docs/en/README.md` should exist.
- `docs/de/README.md` is strongly recommended.
- Update state/channel/device documentation when behavior changes.

## 8) Release Responsibility

- Maintainers execute release scripts after merge.
- PR authors should not pre-release by changing versions/news fields in advance.

## Related Sub-Skills

- [09-submission.md](09-submission.md) — Latest/stable repository submission workflow
- [08-packaging.md](08-packaging.md) — io-package.json and package.json rules
- [10-dev-server.md](10-dev-server.md) — Runtime live validation before release
