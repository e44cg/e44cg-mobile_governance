# Secure Data Pseudonymization — Reference Architecture

**Status:** Draft | **Date:** March 9, 2026 | **Author:** Claude (for Emilio's review)

---

## What this document is

A reusable reference architecture for any project where AI tools (Claude Code, N8N, or similar) need to analyze datasets that contain sensitive client data — without the AI ever seeing personally identifiable information (PII).

Pull this document out at the start of any engagement that involves client data + AI analysis. Work through the design questions in section 2 to adapt it to the specific project.

---

## Important terminology

What this architecture implements is technically **pseudonymization**, not anonymization.

- **Anonymization** means data can *never* be linked back to a person. It's irreversible.
- **Pseudonymization** means PII is replaced with artificial identifiers, but the link *can* be restored under controlled conditions.

Since the whole point of this design is that approved people *can* drill down to see real client identities in dashboards, this is pseudonymization. The distinction matters for compliance frameworks (GDPR, HIPAA, SOC 2) — pseudonymized data is still considered personal data under some regulations, while anonymized data is not.

---

## 1. Architecture overview — the five-layer system

The system has five layers. AI only touches one of them.

| Layer | What it does | Who/what has access | AI involved? |
|-------|-------------|---------------------|:------------:|
| **1. Ingestion** | Raw client data enters the system (database, CSV upload, API feed, etc.) | ETL pipeline only | No |
| **2. PII Vault** | A separate, locked-down store that holds PII fields + a pseudonymous ID for each record | Admin-only access, encrypted at rest, audit-logged | No |
| **3. Tokenization ETL** | SQL scripts that strip PII fields from the dataset, replace them with pseudonymous IDs, and output a clean dataset | Hard-coded SQL — no AI runs this against real data | No* |
| **4. Anonymized Data Store** | The clean dataset with IDs instead of PII — this is the *only* data AI ever sees | AI tools, analysts, workflows | **Yes** |
| **5. Dashboard + Re-ID Service** | Displays AI analysis results; authorized users can resolve IDs back to real client identities when needed | Role-based access, audit-logged | Partial** |

*\* AI may *write* the SQL code for the ETL, but it never *executes* it against real data. See the companion document `etl-review-protocol.md` for the review process.*

*\*\* AI generates the analysis shown in dashboards. The re-identification step (matching an ID back to a real client) is a simple database lookup — no AI involved in that step.*

### How the data flows

```
Raw client data
       │
       ▼
┌──────────────┐
│  Ingestion   │ ── AI cannot see this
└──────┬───────┘
       │
       ├──────────────────┐
       ▼                  ▼
┌──────────────┐   ┌──────────────┐
│   PII Vault  │   │ Tokenization │
│  (locked)    │   │   ETL (SQL)  │ ── AI cannot see this
│              │   └──────┬───────┘
│  Name ↔ ID   │          │
│  Email ↔ ID  │          ▼
│  Phone ↔ ID  │   ┌──────────────┐
└──────────────┘   │  Anonymized  │
       ↑           │  Data Store  │ ── AI works HERE
       │           └──────┬───────┘
       │                  │
       │                  ▼
       │           ┌──────────────┐
       └───────────│  Dashboard   │
    (re-ID lookup  │  + Re-ID     │ ── Approved users only
     for approved  │  Service     │
     users only)   └──────────────┘
```

### The core principle

**AI is confined to one layer.** It can see the anonymized data store and produce analysis. It cannot see, access, or query the PII vault or the raw ingestion layer. Even if the AI tool is compromised, leaked, or misconfigured, only pseudonymous data is exposed.

---

## 2. Design questions to answer per engagement

Before implementing this architecture for a specific project, work through these questions:

### Data inventory

- **What PII fields exist in the dataset?** List exhaustively: names, emails, phone numbers, mailing addresses, account numbers, dates of birth, social security numbers, IP addresses, device identifiers, biometric data, photos, etc.
- **What quasi-identifiers remain after PII removal?** These are fields that aren't PII by themselves but could identify someone in combination — job titles, zip codes, transaction dates, transaction amounts, department names, rare diagnoses, etc. (Example: "the only VP of Marketing in zip code 90210 who made a $47,233 purchase on March 3" is effectively identified even without a name.)
- **How granular does the anonymized data need to be?** Can dates be rounded to months? Can dollar amounts be bucketed into ranges? Can zip codes be truncated to 3 digits? The less granular, the harder to re-identify — but the less useful for analysis.

### ID generation

- **Random UUIDs with a lookup table** — each record gets a random ID; the mapping is stored in the PII vault. Pros: impossible to reverse without the vault. Cons: if the vault is lost, re-identification is impossible.
- **Deterministic hash (e.g., SHA-256 of email + salt)** — the same input always produces the same ID. Pros: consistent across datasets without a central lookup. Cons: if the salt leaks, hashes can be reversed by brute force on known email domains.
- **Recommendation:** Random UUIDs with a lookup table for most cases. Deterministic hashes only if you need to join across independent datasets without a central vault.

### PII vault design

- **Where does the vault live?** Options: separate database, separate schema in the same database (weaker isolation), encrypted file store, dedicated secrets management service.
- **Who has access?** Name the specific roles. "Admin" is not specific enough — which admins, under what conditions?
- **How is access logged?** Every read of the vault should be audit-logged with who, when, what record, and why.
- **What happens if the vault is lost or corrupted?** Is there a backup? Where is the backup stored? Who has access to the backup?

### Re-identification controls

- **Who can resolve IDs back to real clients?** Name specific roles or individuals.
- **Under what conditions?** Always available? Only upon request? Only with manager approval?
- **Is access time-limited?** Can someone get drill-down access for a specific analysis session that expires?
- **What gets logged?** Every re-identification lookup should be audit-logged.

### Compliance

- **What regulations apply?** GDPR, HIPAA, CCPA, SOC 2, client contractual obligations, industry-specific rules.
- **Does the client's contract specify data handling requirements?** Some clients require specific controls, breach notification timelines, or data residency.
- **Is pseudonymized data considered "personal data" under applicable regulations?** (Under GDPR: yes. This affects what you can do with it.)

---

## 3. Implementation approaches

### Option A — Simple (small dataset, spreadsheet-scale)

Best for: one-off analyses, small teams, proof-of-concept work.

- **PII vault:** Encrypted spreadsheet or a separate database table with restricted access credentials
- **ETL:** SQL views or stored procedures that exclude PII columns and replace with UUIDs
- **AI analysis:** Claude or N8N processes exported anonymized CSVs; the AI never connects to the source database
- **Dashboard:** Simple reporting tool; re-identification is manual — an authorized person looks up the ID in the vault when needed
- **Re-ID control:** Physical/procedural — only certain people know the vault password

### Option B — Production (ongoing pipeline, larger scale)

Best for: recurring analyses, multiple analysts, regulated industries, client-facing deliverables.

- **PII vault:** Separate database (or separate cloud account) with encryption at rest, network-level isolation, dedicated credentials
- **ETL:** Scheduled SQL pipeline (dbt, Airflow, or N8N workflow) with automated validation that no PII leaks through (see `etl-review-protocol.md`)
- **AI analysis:** N8N workflows or Claude API calls against the anonymized data store only; connection strings/credentials never include vault access
- **Dashboard:** BI tool (Metabase, Looker, PowerBI, etc.) with row-level security; re-identification via a controlled API endpoint with authentication + audit logging
- **Re-ID control:** Technical — role-based access control on the re-ID endpoint, time-limited tokens, full audit trail

### Hybrid approach

Start with Option A for initial exploration. Move to Option B when the work becomes recurring or client-facing. The architecture is the same — the difference is in how tightly the controls are enforced.

---

## 4. Blast radius analysis

How this architecture limits damage when things go wrong:

| Failure scenario | What's exposed | Blast radius | Notes |
|-----------------|---------------|:------------:|-------|
| AI tool is compromised or leaks data | Anonymized data only — pseudonymous IDs + analytical fields, no names/contacts | **Low** | This is the most likely failure mode, and it's the safest |
| N8N workflow misconfigured, sends data externally | Same — pseudonymous IDs + analytical data only | **Low** | No PII in the data the workflow can access |
| PII vault is breached | Full PII + ID mapping — worst case scenario | **High** | But: smallest attack surface, fewest systems touch it, most tightly controlled |
| ETL bug passes a PII field into the anonymized store | Specific PII field(s) leak into the anonymized store and become accessible to AI | **Medium** | Detectable with automated validation; see `etl-review-protocol.md` |
| Dashboard credentials compromised | Depends on the role — analyst sees anonymized data only; admin might see re-identified data | **Variable** | Role-based access limits exposure |
| Someone with re-ID access misuses it | Specific client identities linked to analytical data | **Medium** | Audit logs create accountability; time-limited access reduces window |

**The architecture's core strength:** The most common and most likely failure modes (AI tool compromised, workflow leaks data) only expose anonymized data. The highest-risk target (PII vault) has the smallest attack surface because the fewest systems and people touch it.

---

## Companion document

See **`etl-review-protocol.md`** for the detailed process of how the ETL/SQL code is written, reviewed, tested, and approved — including the multi-layer AI review protocol that replaces traditional human code review.
