# Security Rules — e44cg Mobile Governance

## Async-first posture

This workspace runs async — Claude works while Emilio is not actively watching.

**Default behavior:** Produce research, plans, and templates. Do not take actions. When in doubt, write a plan and wait.

## Data classification

| Tier | Description | Rules |
|------|-------------|-------|
| **Public** | Templates, planning docs, published patterns | No restrictions |
| **Internal** | Draft plans, research notes, unpublished work | Work freely; don't transmit to external services without approval |
| **Sensitive** | API keys, tokens, credentials, personal data | Never hardcode. Use environment variables. Add to deny rules. |

## Core security rules

1. **No credentials in files.** Ever. Use environment variables or config files excluded by `.gitignore` and `.claudeignore`.
2. **Blast radius principle.** Before any action, ask: "If this goes wrong, is the worst case limited to inconvenience or is it reversible?" If no, write a note and wait for Emilio.
3. **No dependency installs.** This is a markdown workspace. No packages, no build tools.
4. **No network requests without explicit approval.** Explain what data will be sent and where.
5. **No system-level changes.** Do not modify PATH, environment variables, registry, or global configurations.
6. **All output is draft.** Nothing produced here is final until Emilio reviews it in a desktop session.

## Git safety

- Never force push.
- Never amend published commits.
- Commit messages in plain language describing what changed and why.
- Review `git diff` before committing — summarize changes to Emilio in plain language.

## Cross-project isolation

- This workspace cannot edit any Pacific-Outcomes file (hard deny in settings.json).
- This workspace cannot read PO's sensitive paths: banking, taxes, NDA, client originals, `_locked` folders (hard deny).
- General PO files (strategy, outreach) are readable only when Emilio explicitly initiates cross-pollination.
- If work here needs data from Pacific-Outcomes, that is a red flag — stop and discuss with Emilio.

## Mobile repo scoping

- This governance repo can read other e44cg repos for reference (within isolation rules).
- This governance repo should not edit files in non-mobile repos.
- Future `e44cg-mobile-*` repos should only read/edit other `e44cg-mobile-*` repos.

## The `_locked` convention

Any folder named `_locked` is globally denied — no Claude Code session can read or edit its contents.
- Create `_locked` subfolders for personal sensitive data (credentials, private materials).
- The global `settings.json` enforces this. Cannot be overridden.
