# ETL Code Review Protocol — AI-Powered Multi-Layer Review

**Status:** Draft | **Date:** March 9, 2026 | **Author:** Claude (for Emilio's review)
**Companion to:** `secure-data-anonymization.md`

---

## What this document is

The SQL ETL code that strips PII from datasets is the single most critical trust boundary in the pseudonymization architecture. If this code has a bug, PII leaks into the anonymized data store and becomes accessible to AI tools.

This document defines a multi-layer review protocol that uses multiple independent AI reviewers + automated testing to validate the ETL code — arguably more rigorous than a single human reviewer, and practical for a non-coder like Emilio.

---

## Why AI review works well here

The ETL SQL code for pseudonymization is **simple, structural code**. It's not complex algorithms or business logic. It's:

- `SELECT` statements that include some columns and exclude others
- `REPLACE` or `CASE` statements that swap PII values for pseudonymous IDs
- `JOIN` operations to link the ID mapping table
- Optional generalization (e.g., exact ages → age ranges, full zip codes → 3-digit prefixes)

The review question is essentially a checklist: **"Does any PII column survive into the output?"** AI is excellent at checklist tasks — it won't get bored, skip a column, or assume something is fine. And unlike a human reviewer, it can be run independently multiple times.

---

## The five-step review protocol

| Step | Who does it | What they do | Why it matters |
|------|------------|-------------|----------------|
| **1. Author** | AI #1 | Writes the SQL transformation code based on a PII field inventory (the list of fields to strip) | The starting point — AI drafts the code |
| **2. Independent review** | AI #2 (separate session) | Reviews the SQL with one question: "Does any PII column appear in the output? Are quasi-identifiers adequately generalized?" | Catches what the author missed — fresh eyes, no anchoring bias |
| **3. Synthetic data test** | Automated script | Generates fake data with known PII patterns (fake names, emails, phone numbers, SSNs), runs the SQL against it, scans the output for PII leakage using pattern matching (regex for email formats, phone formats, SSN formats, common name lists) | Mechanical verification — catches leaks that code review might miss |
| **4. Test result review** | AI #3 | Reviews the synthetic test results: Did the tests catch everything? Are there edge cases? Any PII patterns in the output that the regex didn't flag? | Belt-and-suspenders — a third perspective on the test results |
| **5. Human approval** | Emilio (or responsible person) | Reviews a plain-language summary: "Here's what PII fields existed. Here's what was stripped. Here's what remains. Here's what the synthetic tests found. Here are any flags." Stamps approval on the *process and results*, not on individual SQL lines. | The human validates the outcome and the protocol, not the code itself |

### What the human summary looks like

The summary presented to Emilio at step 5 should read something like:

> **ETL Review Summary — [Project Name]**
>
> **PII fields in source data:** Name, Email, Phone, Date of Birth, Account Number
>
> **Fields stripped and replaced with IDs:** All five PII fields removed from output, replaced with pseudonymous UUID
>
> **Fields remaining in anonymized output:** Transaction Date, Transaction Amount, Product Category, Region (3-digit zip), Age Range (bucketed)
>
> **Quasi-identifier assessment:** Region + Age Range + Product Category combination — checked, no unique combinations in dataset (k-anonymity ≥ 5)
>
> **Synthetic test results:** 10,000 fake records processed. Output scanned for email patterns, phone patterns, SSN patterns, name matches. **Zero PII detected in output.**
>
> **Flags:** None.
>
> **Recommendation:** Approve for use with real data.

Emilio reads this, checks that the list of stripped fields matches what he knows about the data, and approves. He doesn't need to read SQL.

---

## When this protocol is NOT sufficient

- **Regulated industries with explicit human-code-review requirements.** Some healthcare (HIPAA) and financial (SOC 2 Type II) compliance frameworks may require a qualified human to review the actual code. Check the specific requirements of the applicable framework.
- **Extremely sensitive data** (e.g., protected health information with life-safety implications). Consider adding a qualified third-party code reviewer.
- **Client contracts that specify review procedures.** Some clients require specific review processes — check the contract before relying on AI-only review.

In these cases, the AI review protocol still adds value as a *first pass* — it catches obvious issues before the human reviewer spends time on it. The human review becomes the final step rather than the only step.

---

## Risks and mitigations

### Risk: Quasi-identifier re-identification

Even after stripping all explicit PII, combinations of remaining fields can sometimes uniquely identify a person. Example: "the only 28-year-old in region 902 who bought Product X on March 3" might be identifiable even without a name.

**Mitigation:**
- Generalize quasi-identifiers (exact dates → months, exact amounts → ranges, full zip → 3-digit prefix)
- Run k-anonymity checks: for every combination of quasi-identifiers, are there at least k records that share that combination? (k=5 is a common threshold)
- Include quasi-identifier assessment in the review protocol (step 2 and step 4)

### Risk: PII vault as high-value target

The mapping table (ID → real identity) is the crown jewel. If it's breached, all pseudonymization is undone.

**Mitigation:**
- Store in a separate database/schema from the anonymized data — different credentials, different access controls
- Encrypt at rest
- Network-level isolation where possible (the AI tools should have no network path to the vault)
- Audit-log every read
- Regularly review who has access; remove access that's no longer needed

### Risk: ETL code bugs

A transformation error could pass PII through to the anonymized store. This is the risk the entire review protocol is designed to catch.

**Mitigation:**
- The five-step protocol above
- Automated validation as a recurring check (not just at initial setup): periodically scan the anonymized data store for PII patterns
- If a new column is added to the source data, the ETL code must be re-reviewed — new columns are the most common source of PII leakage

### Risk: Scope creep on dashboard re-identification

If too many people get drill-down access to resolve IDs to real clients, the pseudonymization becomes meaningless in practice.

**Mitigation:**
- Explicit approval process for re-identification access (not just "anyone with a login")
- Time-limited access where possible (access expires after the analysis session)
- Audit trail on every re-identification lookup
- Periodic review of who has access and whether they still need it

### Risk: New data fields added without review

Datasets evolve. A new column added to the source data might contain PII, and if the ETL code doesn't strip it, it flows into the anonymized store.

**Mitigation:**
- Any schema change to the source data triggers a re-run of the review protocol
- Automated alerting when the anonymized data store contains columns not in the approved field list
- Default-deny: the ETL should explicitly list which columns to *include* (whitelist), not which to *exclude* (blacklist). That way, new columns are excluded by default.

---

## Key principle: default-deny field selection

The ETL SQL should be written as a **whitelist**, not a blacklist:

- **Correct (whitelist):** `SELECT transaction_date, amount, category, region FROM source` — only explicitly named fields pass through. New columns are automatically excluded.
- **Wrong (blacklist):** `SELECT * EXCEPT (name, email, phone) FROM source` — new columns pass through by default. A new PII column added to the source would leak.

This single design decision prevents the most common source of PII leakage: schema changes that add new fields.

---

## Summary

The ETL code is the critical trust boundary, but it's also the *simplest* code in the system. The multi-layer AI review protocol (author → independent review → synthetic test → test review → human approval of results) provides rigorous validation without requiring Emilio to read code. The human approves the *process and outcomes*, not the SQL itself.

Combined with the default-deny (whitelist) approach to field selection, this protocol significantly reduces the risk of PII leakage into the anonymized data store.
