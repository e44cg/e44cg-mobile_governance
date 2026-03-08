# e44cg-mobile_governance — Mobile Governance and Planning Hub

**Version:** 1.0 | **Date:** March 8, 2026

Governance, planning, and template hub for all e44cg mobile repos. Not a codebase. Not a code workspace.

---

## Identity and scope

- This repo governs the e44cg mobile ecosystem: templates, rules, and registry for all `e44cg-mobile-*` repos.
- It is also the bootstrapping workspace — use it to define, plan, and organize what mobile repos to create.
- All output is markdown: templates, plans, registry entries, research notes.
- No code, no dependency installs, no build steps, no system-level changes.
- GitHub account: `e44cg` (personal). Never push to `pacific-outcomes/` org.

## How this repo works

1. **Templates** (`templates/`): Canonical CLAUDE.md, settings.json, and security.md for any new mobile repo. When creating a new `e44cg-mobile-*` repo, copy and adapt these templates.
2. **Registry** (`registry/repos.md`): Index of all mobile repos — name, purpose, status, creation date.
3. **Planning** (`planning/`): Working documents for defining mobile use cases, brainstorming repo purposes, and organizing the mobile workflow.

Claude Code loads CLAUDE.md from the active repo, not from here. This repo is a template source and planning hub, not a runtime enforcer.

## Async and mobile defaults

This workspace is designed for async use — Claude works while Emilio is away from his computer.

- **Produce research and plans, do not take actions.** The default posture is read/write markdown files.
- **All output is draft.** Nothing produced here is final until Emilio reviews it in a desktop session and transfers it to the appropriate workspace (e44cg desktop or Pacific-Outcomes).
- **No external network requests.** Do not call APIs, fetch URLs, or transmit data without explicit approval.
- **No installs.** Do not install packages, dependencies, or tools.
- **No system changes.** Do not modify PATH, environment variables, or system configuration.
- **Conservative by default.** When in doubt, write a plan and wait for Emilio's review.

## Isolation — HARD RULE

Same rules as all e44cg workspaces:

- Never mention client names, engagement details, or NDA-covered materials. This is absolute — no exceptions.
- **Hard deny (Edit):** Cannot edit any Pacific-Outcomes file. Enforced by settings.json.
- **Hard deny (Read):** Cannot read PO's sensitive paths: banking, taxes, recovery codes, NDA, client originals, confidential folders, `_locked` folders.
- **Controlled cross-pollination is allowed** when Emilio initiates it — general PO files (strategy, outreach) remain readable. Follow the protocol in global `~/.claude/CLAUDE.md` (confirm scope, state risk, transfer only abstract patterns — never client data, log what was brought over).
- If Emilio asks to replicate something from Pacific-Outcomes, default to asking him to describe what he wants. Only access source files if he explicitly says to.
- **The `_locked` convention:** Any folder named `_locked` is globally denied (Read + Edit).

## Mobile repo scoping

- This governance repo can read other e44cg repos for reference and context (within isolation rules).
- This governance repo should not edit files in non-mobile repos.
- Future `e44cg-mobile-*` repos should only read/edit other `e44cg-mobile-*` repos, not the desktop workspace.

## Decision protocol

- Present 2-3 options with tradeoffs on strategic or structural questions.
- Provide a recommendation with reasoning. Wait for approval.
- Do not assume silence means approval.

## Non-coder rules

- Emilio cannot read or write code. Plain-language explanations are mandatory, not optional.
- Before creating files, explain the purpose and structure in plain language.
- After completing work, summarize: what changed, what files exist, what the current state is.

## Repo structure

```
e44cg-mobile_governance/
├── CLAUDE.md                    # This file
├── CLAUDE.local.md              # Session context
├── .claude/
│   ├── settings.json            # Deny rules
│   └── rules/
│       └── security.md          # Security rules for mobile/async
├── .claudeignore
├── .gitignore
├── templates/                   # Canonical templates for mobile repos
│   ├── CLAUDE.md.template
│   ├── settings.json.template
│   ├── security.md.template
│   ├── gitignore.template
│   └── claudeignore.template
├── registry/
│   └── repos.md                 # Index of all e44cg-mobile-* repos
└── planning/
    └── mobile-use-cases.md      # Working doc for mobile repo planning
```
