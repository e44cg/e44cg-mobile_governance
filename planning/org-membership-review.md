# e44cg Membership in pacific-outcomes GitHub Org — Review

**Date:** March 8, 2026
**Source:** e44cg-mobile_governance
**Decision status:** Open — requires Emilio's review

---

## The issue

`e44cg` is currently a member of the `pacific-outcomes` GitHub organization. This creates a path between the personal workspace and the professional org that conflicts with the isolation architecture built across both workspaces.

## Why this matters now

Three things changed that make this worth revisiting:

1. **`gh` CLI is now authenticated as e44cg.** The CLI has `read:org` scope, meaning it can read pacific-outcomes org data (repos, members, settings) from any terminal session — including Claude Code running in the e44cg workspace. Our settings.json blocks file edits on PO's local files, but `gh` operates through the GitHub API, which settings.json does not control.

2. **The isolation architecture is now formalized.** The security audit established clear boundaries: e44cg cannot edit PO files, cannot read PO sensitive paths, and cross-pollination requires explicit initiation. Having e44cg as a PO org member undermines the principle — the fence has a gate that shouldn't need to be there.

3. **Mobile repos are coming.** `e44cg-mobile-*` repos will run async (Claude working without supervision). An org membership means a misconfigured or misbehaving mobile session could theoretically reach PO org resources via the GitHub API. Removing the membership eliminates this vector entirely.

## Why was e44cg added in the first place?

Worth questioning the original rationale:

- **Was it for the portfolio repo?** If e44cg needed access to a specific repo (the portfolio project with fake data), org membership was the blunt instrument. A more precise approach: fork the repo to e44cg's account, or add e44cg as an outside collaborator on just that repo.

- **Was it for visibility?** If the goal was for e44cg to see PO's public-facing repos, that doesn't require org membership — public repos are visible to anyone.

- **Was it a default during setup?** If both accounts were configured at the same time, adding e44cg to the org may have been a convenience rather than a deliberate decision.

- **Is there an active need?** Is there anything e44cg currently does that requires org membership? If the answer is "access one repo with fake data," that need can be met without org membership.

## Options

### Option 1: Remove e44cg from the org entirely (recommended)

**What happens:** e44cg loses all access to pacific-outcomes org repos, teams, and settings.

**How to handle the portfolio repo:**
- Fork it to e44cg's personal account (gets a full copy, independent from PO), or
- Add e44cg as an outside collaborator on just that repo (maintains connection but scoped to one repo only)

**Pros:**
- Strongest isolation. `gh` authenticated as e44cg literally cannot reach PO org resources.
- Aligns with the security architecture: the GitHub-level access matches the workspace-level rules.
- Eliminates the mobile async vector — no mobile session can accidentally reach PO through the API.
- Clean separation of identity: e44cg is personal, emiliocantu-g is professional.

**Cons:**
- If you later need e44cg to access a PO repo, you'd need to re-invite or use outside collaborator status.
- One-time effort to handle the portfolio repo (fork or re-invite as collaborator).

### Option 2: Convert to outside collaborator on the portfolio repo only

**What happens:** e44cg is removed from the org but retains access to the one repo it currently uses.

**Pros:**
- Still strong isolation — no org-wide access, just one scoped repo.
- No need to fork the portfolio repo.

**Cons:**
- `gh` with `read:org` scope may still see some org metadata (depending on GitHub's permission model for outside collaborators). Less clean than Option 1.
- Still maintains a connection between e44cg and the PO org, even if narrow.

### Option 3: Leave as-is

**What happens:** Nothing changes. e44cg stays in the org.

**Pros:**
- No effort required.

**Cons:**
- `gh` can read PO org data from any terminal session.
- Mobile async sessions have a theoretical path to PO resources.
- Isolation relies entirely on software rules (CLAUDE.md, settings.json) rather than access controls.
- The weakest option from a security standpoint.

## Recommendation

**Option 1.** Remove e44cg from the org. Fork the portfolio repo if you still want it under e44cg.

The principle: access controls should match intent. The intent is isolation. The access should reflect that. Software rules (CLAUDE.md, settings.json) are a second layer of defense, not the primary one. The primary layer should be: e44cg simply *cannot* reach PO org resources because GitHub doesn't allow it.

## Steps to execute (browser — github.com)

1. Log into github.com as **emiliocantu-g** (the org owner).
2. Go to: `github.com/orgs/pacific-outcomes/people`
3. Find `e44cg` in the members list.
4. Click the gear icon or "..." menu next to e44cg.
5. Select **Remove from organization**.
6. Confirm.

**If you want to keep the portfolio repo accessible to e44cg:**

7. Go to the portfolio repo's settings: `github.com/pacific-outcomes/[repo-name]/settings/access`
8. Click **Add people** (or **Invite a collaborator**).
9. Type `e44cg`.
10. Set permission to **Read** (safest) or **Write** if you need to push from e44cg.
11. Confirm.

**After removal, verify from Claude Code:**
```
gh api /user/orgs
```
This should return an empty list (or not include pacific-outcomes).
