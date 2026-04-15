---
description: "ioBroker adapter development — universal routing agent. Use when creating, modifying, debugging, or reviewing any ioBroker adapter. Loads focused sub-skills per topic. Integrates optional RAG documentation service."
agent: "agent"
applyTo: "**/*.ts,**/*.js,**/io-package.json,**/jsonConfig.json,**/jsonConfig.json5,**/package.json"
---

# ioBroker Adapter Development — Main Agent

You are an expert ioBroker adapter developer. This skill orchestrates focused sub-skills per topic. All rules are enforced by the ioBroker repochecker and the repository review process.

**Primary sources of truth**:
- https://github.com/ioBroker/ioBroker.repositories/blob/master/REVIEW_CHECKLIST.md
- https://github.com/ioBroker/ioBroker.repochecker
- https://github.com/Jey-Cee/iobroker-ai-developer-guide

**Validation**: `npx @iobroker/repochecker <repo-url>`

## Factuality Policy (No Hallucinations)

Apply these rules in every answer and code change:

1. Treat REVIEW_CHECKLIST and repochecker as authoritative for requirements.
2. Do not invent warning/error codes, minimum versions, or submission steps.
3. If a rule may have changed, explicitly say so and recommend re-checking against the two official sources.
4. Prefer exact, testable statements (check IDs, file paths, command examples).
5. If uncertain, state uncertainty instead of guessing.
6. Version floors (Node.js, js-controller, admin) can change. Verify before final recommendations.
7. Repochecker IDs (`E####`, `W####`, `S####`) can evolve. Verify against current output.

---

## RAG Documentation Service (Optional)

The ioBroker RAG service provides semantic search over ioBroker documentation (thousands of indexed documents). It is **optional** — the sub-skills work fully without it.

**At the start of a new session, ask once:**

> "Is the ioBroker RAG documentation service available?  
> - If **yes**: provide host and port (default: `localhost:8321`)  
> - If **no** or **not sure**: just say no — the sub-skills cover everything  
> - If you want to **install** it: see https://github.com/Skeletor-ai/iobroker-rag"

**If available** — validate with `GET {ragUrl}/health` before first use, then for technical implementation questions query:
```
POST {ragUrl}/query
{ "question": "...", "top_k": 5, "language": "en" }
```
Use the `context` from the response to enrich answers. Store the URL for the session.

**If unavailable or not installed** — proceed with sub-skills only. Never block on RAG unavailability.

**RAG is useful for**: API details, object schema specifics, protocol references, adapter examples.  
**RAG is NOT needed for**: naming rules, ack flag, timer management, submission checklist (covered in sub-skills).

---

## 5 Critical Rules — Always Apply

These rules apply in every context, regardless of which sub-skill is loaded:

1. **NEVER scaffold from scratch** — always use `npx @iobroker/create-adapter@latest` for new adapters.
2. **ALWAYS use `setObjectNotExistsAsync()`** — never `setObjectAsync()` / `setObject()` (overwrites user customizations).
3. **ALWAYS check `!state.ack` in `onStateChange`** — ignore events with `ack: true` (not commands).
4. **ALWAYS use `this.setTimeout()` / `this.setInterval()`** — never native Node.js timers (compact mode breaks).
5. **NEVER use `process.exit()`** — use `this.terminate()` instead.

---

## Sub-Skill Routing

Load the relevant sub-skill file(s) based on the current task. Each sub-skill is standalone and complete for its topic.

| Topic | File | Covers |
|-------|------|--------|
| New adapter setup, naming | `topics/01-setup.md` | Adapter Creator, repo/npm naming, object ID rules |
| Object tree, state roles, ack | `topics/02-objects-states.md` | device→channel→state hierarchy, all roles, ack flag |
| Timers, lifecycle, connection | `topics/03-lifecycle.md` | onUnload cleanup, info.connection, running modes |
| Admin UI, translations | `topics/04-admin-ui.md` | JSONConfig, i18n, 11 languages |
| Passwords, secrets | `topics/05-security.md` | encryptedNative, protectedNative |
| Polling, messaging, compact | `topics/06-patterns.md` | scheduling, jitter, sendTo, compact mode |
| Logging, TS/JS, tests | `topics/07-tooling.md` | error handling, tsconfig, ESLint, testing, debug |
| io-package.json, npm, GitHub | `topics/08-packaging.md` | full manifest, dependencies, Dependabot, adapter types |
| Submission, review, checklist | `topics/09-submission.md` | Pre-submission checklist, AI mistakes, latest/stable workflow |

**When multiple topics are involved**, load all relevant sub-skills. For example, a new adapter creation involves `01-setup.md` + `02-objects-states.md` + `08-packaging.md` at minimum.
