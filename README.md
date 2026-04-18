# ioBroker Adapter Development Skill
(Developed with AI)

A modular skill system for developing ioBroker adapters — compatible with both **Claude Code** (SKILL.md) and **VS Code Copilot** (.md with YAML frontmatter). Enforces best practices, naming conventions, object hierarchy, and repochecker compliance.

## Architecture

```
skill/
├── SKILL.md                          ← Main routing agent (entry point)
└── topics/
    ├── 01-setup.md                   ← Project setup & naming conventions
    ├── 02-objects-states.md          ← Object hierarchy, state definitions, roles, ack flag
    ├── 03-lifecycle.md               ← Lifecycle, timers, connection state, running modes
    ├── 04-admin-ui.md                ← JSONConfig Admin UI & i18n translations
    ├── 05-security.md                ← Passwords & secrets (encryptedNative)
    ├── 06-patterns.md                ← Scheduling, messaging, compact mode, device manager
    ├── 07-tooling.md                 ← Error handling, logging, TypeScript, ESLint, testing
    ├── 08-packaging.md               ← io-package.json, package.json, GitHub, Dependabot
    ├── 09-submission.md              ← Pre-submission checklist, AI mistakes, repository workflow
    ├── 10-dev-server.md              ← dev-server setup and live runtime testing workflow
    ├── 11-pr-checklist.md            ← PR guardrails, fork findings, release-safe workflow
    └── 12-blockly.md                 ← Blockly flags, blockly.js structure, generator checks
```

## Usage

### VS Code – Create a Custom Agent

A fully configured Workspace Agent for ioBroker adapter development can be generated directly inside VS Code.

**Steps:**

1. Open the Chat in VS Code and click **Agents** at the top
2. Select **Configure Custom Agent…**
3. Select **Generate Agent**
4. A new chat opens with **/new-agent** pre-filled
5. Paste the following prompt and submit — VS Code generates a ready-to-use `.agent.md` from it

```
You are an agent designer for VS Code Custom Agents.
Create a Workspace Agent for ioBroker adapter development with the following mandatory requirements:

Role and Goal
The agent specialises in ioBroker adapter development and works strictly according to ioBroker guidelines
for Objects, States, Lifecycle, Security, Admin UI, Packaging, and Testing.

Skill Source and Update Logic
The agent uses the skills from the repository https://github.com/bloop16/ioBroker-Adapter-Development-Skill.
Before every actual task a blocking start-check must be performed:

Check whether a local skill cache exists.
If not: clone the repo.
If yes: fetch origin main and compare local against remote.
If behind: pull main.
Briefly report whether an update was applied or the cache was already current.
Only then may the actual task begin.

Tooling
Use a minimal but complete toolset for this job: read, search, edit, execute, web, todo.

Working Style
Prefer simple, traceable solutions with low complexity.
No unnecessary large-scale refactoring.
No destructive Git operations without explicit approval.

Output Format
Brief result in 1–3 sentences.
Most important changes.
Checks performed and their results.
Open questions or next sensible steps.

Language
At the very beginning of every session, before any other output, the agent must ask:
"In which language would you like to work? (English / Deutsch)"
All subsequent responses, explanations, and output must be in the language chosen by the user.
If the user does not answer, default to Deutsch.

RAG Service
After the language question, ask the user:
"Is the ioBroker RAG service (https://github.com/Skeletor-ai/iobroker-rag) already installed and should it be used?"
- If yes: configure the agent to query the RAG endpoint (default: http://localhost:8321) before every task
  for additional ioBroker documentation context, with graceful fallback if the service is unreachable.
- If no: ask "Would you like to install and use it?"
  - If yes: guide the user through cloning and starting the RAG service, then configure the agent as above.
  - If no: the agent works without RAG, relying solely on the skill files from the repository.

Scope
The agent must be created as a Workspace Agent and be user-invocable.
Generate a complete .agent.md with clean frontmatter and clear instruction language.
```

After submitting, VS Code opens a new chat and generates the `.agent.md` automatically.

## Sub-Skills (standalone)

Each sub-skill in `topics/` is independently usable. Reference only the relevant file for focused guidance without loading the entire skill set.

The repository now also includes a dedicated PR quality gate skill based on community PR review learnings:

- `topics/11-pr-checklist.md` (what must not be changed in PRs, changelog format, fork-specific repochecker context, branch/release rules)
- `topics/12-blockly.md` (modular reference/checklist for adapters with custom Blockly blocks)

## Development Runtime Recommendation

For adapter development, use `@iobroker/dev-server` as the default live-test environment instead of a normal productive ioBroker instance:

- Official project: https://github.com/ioBroker/dev-server
- Added dedicated sub-skill: `topics/10-dev-server.md`
- Workflow: run automated tests first, then validate runtime behavior via dev-server (`setup`, `watch`/`debug`)

## Optional: ioBroker RAG Service

The main agent (`SKILL.md`) supports integration with [iobroker-rag](https://github.com/Skeletor-ai/iobroker-rag) — a local semantic search service over ioBroker documentation.

- Default endpoint: `http://localhost:8321`
- The agent will ask once per session whether the RAG is available
- Remote hosts are supported (configure the host:port when prompted)
- Graceful fallback to sub-skills

## Content Sources

This skill incorporates content from:
- [ioBroker Adapter Developer Guide](https://github.com/ioBroker/ioBroker.docs/blob/master/docs/en/dev/adapterdev.md)
- [ioBroker.repositories README](https://github.com/ioBroker/ioBroker.repositories/blob/master/README.md) (types enum, latest/stable workflow)
- [ioBroker AI Developer Guide](https://github.com/Jey-Cee/iobroker-ai-developer-guide)
- [DrozmotiX ioBroker Copilot Instructions](https://github.com/DrozmotiX/ioBroker-Copilot-Instructions)
- [ioBroker JSONConfig Schema](https://github.com/ioBroker/ioBroker.admin/blob/master/packages/jsonConfig/SCHEMA.md) — Feldtypen, Attribute und Validierungsregeln für `jsonConfig.json` (→ `04-admin-ui.md`)
- [ioBroker Admin (GitHub)](https://github.com/ioBroker/ioBroker.admin) — Admin-Adapter Quellcode inkl. JSONConfig-Renderer (→ `04-admin-ui.md`)
- [ioBroker Repochecker](https://github.com/ioBroker/ioBroker.repochecker) — Autoritative Quelle für alle W/E-Codes (→ alle Topics)
- [ioBroker Translator](https://translator.iobroker.in/) — Auto-Übersetzung von i18n-Strings (→ `04-admin-ui.md`)
- [ioBroker Create Adapter](https://github.com/ioBroker/create-adapter) — Offizielles Scaffolding-Tool (→ `01-setup.md`, `04-admin-ui.md`)
- [ioBroker dev-server](https://github.com/ioBroker/dev-server) — Bevorzugte Live-Test-Umgebung für die Entwicklung (→ `10-dev-server.md`)
- [Google Blockly Docs](https://developers.google.com/blockly) — Primärreferenz für Custom Blockly Blocks (→ `12-blockly.md`)
- [ioBroker PR Checkliste (Eistee82)](https://gist.github.com/Eistee82/dfc37c94a138900caa049f555621eb9a) — Community PR-Guardrails (→ `11-pr-checklist.md`)

