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
    └── 09-submission.md              ← Pre-submission checklist, AI mistakes, repository workflow
```

## Usage

### Claude Code
Add `skill/SKILL.md` to your project's `.claude/` directory or reference it directly. The routing agent auto-discovers the RAG service (optional) and loads the appropriate sub-skill for each task.

### VS Code Copilot
Reference individual topic files in your Copilot instructions or attach them to a conversation. Each file has YAML frontmatter with `applyTo` patterns for automatic activation.

```yaml
# Example: skill/topics/02-objects-states.md is automatically applied to
# all .ts and .js files via its frontmatter:
# applyTo: "**/*.ts,**/*.js"
```

## Sub-Skills (standalone)

Each sub-skill in `topics/` is independently usable. Reference only the relevant file for focused guidance without loading the entire skill set.

## Optional: ioBroker RAG Service

The main agent (`SKILL.md`) supports integration with [iobroker-rag](https://github.com/Skeletor-ai/iobroker-rag) — a local semantic search service over ioBroker documentation.

- Default endpoint: `http://localhost:8321`
- The agent will ask once per session whether the RAG is available
- Remote hosts are supported (configure the host:port when prompted)
- Graceful fallback to sub-skills only if not available

## Content Sources

This skill incorporates content from:
- [ioBroker Adapter Developer Guide](https://github.com/ioBroker/ioBroker.docs/blob/master/docs/en/dev/adapterdev.md)
- [ioBroker.repositories README](https://github.com/ioBroker/ioBroker.repositories/blob/master/README.md) (types enum, latest/stable workflow)
- [ioBroker AI Developer Guide](https://github.com/Jey-Cee/iobroker-ai-developer-guide)
- [DrozmotiX ioBroker Copilot Instructions](https://github.com/DrozmotiX/ioBroker-Copilot-Instructions)

