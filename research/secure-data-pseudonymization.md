# Secure Data Pseudonymization — Reference Architecture

**Status:** Draft | **Date:** March 9, 2026 | **Author:** Claude (for Emilio's review)

---

## What this document is

A reusable reference architecture for any project where AI tools (Claude Code, N8N, or similar) need to analyze datasets that contain sensitive client data — without the AI ever seeing personally identifiable information (PII).

Pull this document out at the start of any engagement that involves client data + AI analysis. Work through the design questions in section 3 to adapt it to the specific project.

---

## 1. Terminology — pseudonymization vs. anonymization

What this architecture implements is **pseudonymization**, not anonymization. The distinction is not academic — it has direct legal and practical consequences.

- **Anonymization** means data can *never* be linked back to a person. It's irreversible. Under GDPR, truly anonymized data is no longer considered personal data and falls outside the regulation entirely (GDPR Recital 26).
- **Pseudonymization** means PII is replaced with artificial identifiers, but the link *can* be restored under controlled conditions. Under GDPR, pseudonymized data **is still personal data** in the hands of anyone who could access the re-identification key (EDPB Guidelines 01/2025, Section 2.1).

Since the whole point of this design is that approved people *can* drill down to see real client identities in dashboards, this is pseudonymization. That means:
- GDPR obligations still apply to the pseudonymized dataset
- The re-identification key (the PII vault) must be stored separately with its own access controls
- Technical and organizational measures (TOMs) must prevent unauthorized re-identification

The EDPB's January 2025 guidelines formalize a three-step pseudonymization process: (1) transform data by removing/replacing identifiers, (2) store the re-identification key separately with independent protections, and (3) implement TOMs against unauthorized re-identification [1]. This architecture follows all three steps.

**However:** Pseudonymized data in the hands of a party that does *not* have access to the re-identification key — and has no other means of identifying individuals — may be treated as anonymous data under GDPR [1]. This is directly relevant to the AI layer in this architecture: the AI tools have no access to the PII vault, so from the AI's perspective, the data it processes may qualify as anonymous. This is the strongest legal argument for the architecture's safety.

---

## 2. Architecture overview — the five-layer system

The system has five layers. AI only touches one of them.

| Layer | What it does | Who/what has access | AI involved? |
|-------|-------------|---------------------|:------------:|
| **1. Ingestion** | Raw client data enters the system (database, CSV upload, API feed, etc.) | ETL pipeline only | No |
| **2. PII Vault** | A separate, locked-down store that holds PII fields + a pseudonymous ID for each record | Admin-only access, encrypted at rest, audit-logged | No |
| **3. Tokenization ETL** | SQL scripts that strip PII fields from the dataset, replace them with pseudonymous IDs, and output a clean dataset | Hard-coded SQL — no AI runs this against real data | No* |
| **4. Pseudonymized Data Store** | The clean dataset with IDs instead of PII — this is the *only* data AI ever sees | AI tools, analysts, workflows | **Yes** |
| **5. Dashboard + Re-ID Service** | Displays AI analysis results; authorized users can resolve IDs back to real client identities when needed | Role-based access, audit-logged | Partial** |

*\* AI may *write* the SQL code for the ETL, but it never *executes* it against real data. See the companion document `ai-review-protocol.md` for the review process.*

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
└──────────────┘   │ Pseudonymized│
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

**AI is confined to one layer.** It can see the pseudonymized data store and produce analysis. It cannot see, access, or query the PII vault or the raw ingestion layer. Even if the AI tool is compromised, leaked, or misconfigured, only pseudonymous data is exposed.

This principle — separating the data processor from the identifier — aligns with what NIST SP 800-188 calls "structural de-identification": using system architecture, not just data transformation, to enforce privacy boundaries [2]. The AI tools have no network path, no credentials, and no logical access to the PII vault. The separation is enforced at the infrastructure level, not just the application level.

---

## 3. Design questions to answer per engagement

Before implementing this architecture for a specific project, work through these questions:

### Data inventory

- **What PII fields exist in the dataset?** List exhaustively: names, emails, phone numbers, mailing addresses, account numbers, dates of birth, social security numbers, IP addresses, device identifiers, biometric data, photos, etc. NIST SP 800-188 provides a detailed taxonomy of identifier types to use as a checklist [2].
- **What quasi-identifiers remain after PII removal?** These are fields that aren't PII by themselves but could identify someone in combination — job titles, zip codes, transaction dates, transaction amounts, department names, rare diagnoses, etc.

  **This is the most underestimated risk.** Latanya Sweeney's foundational 1997 research demonstrated that 87% of Americans could be uniquely identified using just three quasi-identifiers: ZIP code, birth date, and gender [3]. She proved this by re-identifying the Governor of Massachusetts in a "de-identified" medical dataset using a $20 voter registration database. This work led directly to the development of k-anonymity as a formal privacy model and shaped HIPAA's de-identification standards.

- **How granular does the pseudonymized data need to be?** Can dates be rounded to months? Can dollar amounts be bucketed into ranges? Can zip codes be truncated to 3 digits? The less granular, the harder to re-identify — but the less useful for analysis. This tradeoff between privacy and utility is central to all de-identification work [2].

### Privacy models for quasi-identifiers

Once quasi-identifiers are identified, apply formal privacy models to evaluate residual risk:

- **k-Anonymity** (Sweeney, 2002): Every record in the dataset must be indistinguishable from at least k-1 other records based on quasi-identifier values. If k=5, any combination of quasi-identifiers matches at least 5 people [3]. Limitation: doesn't protect against attacks when the sensitive attribute has low diversity within a group.

- **l-Diversity** (Machanavajjhala et al., 2006): Extends k-anonymity by requiring that each equivalence group contains at least l "well-represented" values for the sensitive attribute [4]. Prevents the "homogeneity attack" where all records in a k-anonymous group share the same sensitive value.

- **t-Closeness** (Li et al., 2007): Requires that the distribution of a sensitive attribute within any quasi-identifier group is within distance t of the attribute's distribution in the full dataset, measured by Earth Mover's Distance [5]. Prevents the "skewness attack" where the distribution within a group reveals information even when l-diversity is satisfied.

- **Differential Privacy** (Dwork, 2006): A mathematical framework that adds calibrated noise to query results, providing formal guarantees that any individual's presence or absence in the dataset cannot be detected. Used at scale by Apple (keyboard analytics, emoji usage), Google (Chrome Privacy Sandbox), and the US Census Bureau [6]. More complex to implement than the models above, but offers the strongest theoretical guarantees. NIST SP 800-188 notes that while differential privacy has matured significantly, it is "still not sufficiently mature to be used by many government agencies" for general-purpose de-identification [2].

**Practical recommendation:** For most consulting-scale work, start with k-anonymity (k ≥ 5) as the baseline check. It's the simplest to understand, explain to clients, and verify. Add l-diversity or t-closeness if the dataset contains sensitive attributes (health conditions, financial status) where the homogeneity attack is a real concern. Consider differential privacy only for large-scale, repeated-query scenarios.

The open-source **ARX Data Anonymization Tool** supports all of these models and can compute k-anonymity and l-diversity metrics on your datasets. Google Cloud's Sensitive Data Protection also offers built-in k-anonymity and l-diversity analysis [7].

### ID generation

- **Random UUIDs with a lookup table** — each record gets a random ID; the mapping is stored in the PII vault. Pros: impossible to reverse without the vault. Cons: if the vault is lost, re-identification is impossible.
- **Deterministic hash (e.g., SHA-256 of email + salt)** — the same input always produces the same ID. Pros: consistent across datasets without a central lookup. Cons: if the salt leaks, hashes can be reversed by brute force on known email domains.
- **Cryptographic pseudonyms (e.g., HMAC-based or encryption-based)** — the EDPB guidelines specifically mention message authentication codes (MACs) and encryption algorithms as valid pseudonymization methods, alongside lookup tables [1].
- **Recommendation:** Random UUIDs with a lookup table for most cases. The EDPB guidelines note that random pseudonym generation provides the strongest protection because there is no algorithmic relationship between the pseudonym and the original identifier [1]. Deterministic hashes only if you need to join across independent datasets without a central vault.

### PII vault design

- **Where does the vault live?** Options: separate database, separate schema in the same database (weaker isolation), encrypted file store, dedicated secrets management service. The EDPB guidelines emphasize that the "additional information" (the re-identification key) must be kept separately and subject to independent technical and organizational measures [1].
- **Who has access?** Name the specific roles. "Admin" is not specific enough — which admins, under what conditions?
- **How is access logged?** Every read of the vault should be audit-logged with who, when, what record, and why.
- **What happens if the vault is lost or corrupted?** Is there a backup? Where is the backup stored? Who has access to the backup?
- **Network isolation:** The AI tools should have no network path to the vault. This is the strongest form of separation — even a compromised AI tool cannot reach the PII vault if the network doesn't allow it.

### Re-identification controls

- **Who can resolve IDs back to real clients?** Name specific roles or individuals.
- **Under what conditions?** Always available? Only upon request? Only with manager approval?
- **Is access time-limited?** Can someone get drill-down access for a specific analysis session that expires?
- **What gets logged?** Every re-identification lookup should be audit-logged. The EDPB guidelines require that any re-identification attempt — whether authorized or not — be detectable through technical measures [1].

### Compliance

- **What regulations apply?** GDPR, HIPAA, CCPA, SOC 2, client contractual obligations, industry-specific rules.
- **Does the client's contract specify data handling requirements?** Some clients require specific controls, breach notification timelines, or data residency.
- **Is pseudonymized data considered "personal data" under applicable regulations?** Under GDPR: yes, for anyone who could access the re-identification key. Under HIPAA: pseudonymized data may qualify as a "limited data set" which has its own rules. Check the specific framework.
- **Data Protection Impact Assessment (DPIA):** Under GDPR, a DPIA may be required when pseudonymization is part of high-risk processing. Document the techniques used, decisions made, and risk assessments performed [1].

---

## 4. Implementation approaches

### Option A — Simple (small dataset, spreadsheet-scale)

Best for: one-off analyses, small teams, proof-of-concept work.

- **PII vault:** Encrypted spreadsheet or a separate database table with restricted access credentials
- **ETL:** SQL views or stored procedures that exclude PII columns and replace with UUIDs
- **AI analysis:** Claude or N8N processes exported pseudonymized CSVs; the AI never connects to the source database
- **Dashboard:** Simple reporting tool; re-identification is manual — an authorized person looks up the ID in the vault when needed
- **Re-ID control:** Physical/procedural — only certain people know the vault password
- **Privacy model:** Manual k-anonymity check (review the dataset for unique quasi-identifier combinations)

### Option B — Production (ongoing pipeline, larger scale)

Best for: recurring analyses, multiple analysts, regulated industries, client-facing deliverables.

- **PII vault:** Separate database (or separate cloud account) with encryption at rest, network-level isolation, dedicated credentials
- **ETL:** Scheduled SQL pipeline (dbt, Airflow, or N8N workflow) with automated validation that no PII leaks through (see `ai-review-protocol.md`)
- **AI analysis:** N8N workflows or Claude API calls against the pseudonymized data store only; connection strings/credentials never include vault access
- **Dashboard:** BI tool (Metabase, Looker, PowerBI, etc.) with row-level security; re-identification via a controlled API endpoint with authentication + audit logging
- **Re-ID control:** Technical — role-based access control on the re-ID endpoint, time-limited tokens, full audit trail
- **Privacy model:** Automated k-anonymity/l-diversity checks using ARX or equivalent tooling, run as part of the ETL pipeline
- **PII scanning:** Automated PII detection on the pseudonymized data store using pattern matching (regex for emails, SSNs, phone formats) and/or LLM-powered classification — run on a recurring schedule, not just at setup. Databricks' LogSentinel system demonstrates this approach at scale, achieving 92% precision and 95% recall for PII detection with automated remediation ticket creation [8].

### Hybrid approach

Start with Option A for initial exploration. Move to Option B when the work becomes recurring or client-facing. The architecture is the same — the difference is in how tightly the controls are enforced.

---

## 5. Blast radius analysis

How this architecture limits damage when things go wrong:

| Failure scenario | What's exposed | Blast radius | Notes |
|-----------------|---------------|:------------:|-------|
| AI tool is compromised or leaks data | Pseudonymized data only — pseudonymous IDs + analytical fields, no names/contacts | **Low** | This is the most likely failure mode, and it's the safest |
| N8N workflow misconfigured, sends data externally | Same — pseudonymous IDs + analytical data only | **Low** | No PII in the data the workflow can access |
| PII vault is breached | Full PII + ID mapping — worst case scenario | **High** | But: smallest attack surface, fewest systems touch it, most tightly controlled |
| ETL bug passes a PII field into the pseudonymized store | Specific PII field(s) leak into the pseudonymized store and become accessible to AI | **Medium** | Detectable with automated validation; see `ai-review-protocol.md` |
| Dashboard credentials compromised | Depends on the role — analyst sees pseudonymized data only; admin might see re-identified data | **Variable** | Role-based access limits exposure |
| Someone with re-ID access misuses it | Specific client identities linked to analytical data | **Medium** | Audit logs create accountability; time-limited access reduces window |
| Quasi-identifier re-identification (no breach needed) | Individuals identified through unique combinations of non-PII fields | **Medium** | Mitigated by k-anonymity checks and generalization; hardest to detect after the fact |

**The architecture's core strength:** The most common and most likely failure modes (AI tool compromised, workflow leaks data) only expose pseudonymized data. The highest-risk target (PII vault) has the smallest attack surface because the fewest systems and people touch it.

The re-identification risk from quasi-identifiers is the most subtle threat. Unlike the other failure modes, it doesn't require a breach — it requires only that someone combines the pseudonymized data with an external dataset (as Sweeney demonstrated with the voter registration database [3]). This is why the k-anonymity checks and quasi-identifier generalization in section 3 are not optional nice-to-haves — they're a core part of the architecture.

---

## 6. Related patterns and adjacent use cases

This pseudonymization architecture is one approach within a broader family. Depending on the engagement, alternative or complementary approaches may be more appropriate:

- **Synthetic data generation:** Instead of pseudonymizing real data, generate entirely artificial data that preserves the statistical properties of the original. Useful for development/testing environments and for sharing datasets externally. Does not require a PII vault because no real data exists. Limitation: subtle correlations in the real data may be lost or distorted.
- **Federated learning / local differential privacy:** The data never leaves the source — the AI model trains locally and only sends aggregated, noisy gradients. Used by Apple for keyboard analytics and emoji trends across hundreds of millions of devices [6]. Appropriate when you need to learn from distributed data without centralizing it.
- **Secure multi-party computation (MPC):** Multiple parties compute a joint result without any party seeing another's raw data. Appropriate for cross-organizational analysis (e.g., benchmarking across competing firms).
- **Data enclaves / trusted execution environments:** Raw data stays in a controlled environment; approved queries can be run against it, but the data itself cannot be extracted. Used by some government statistical agencies and healthcare research consortia.

The pseudonymization architecture in this document occupies a practical middle ground: simpler to implement than federated learning or MPC, but more robust than basic de-identification. It's best suited for scenarios where one organization controls the data and wants to let AI tools analyze it safely.

---

## Companion document

See **`ai-review-protocol.md`** for the detailed process of how code is written, reviewed, tested, and approved using multi-layer AI review — including the ETL trust boundary and other applicable scenarios.

---

## Sources

[1] European Data Protection Board (EDPB). "Guidelines 01/2025 on Pseudonymisation." January 16, 2025. https://www.edpb.europa.eu/system/files/2025-01/edpb_guidelines_202501_pseudonymisation_en.pdf — The first formal regulatory guidance on pseudonymization under GDPR. Defines the three-step process (transform, store separately, implement TOMs), clarifies that pseudonymized data remains personal data for the controller, and specifies when a recipient without the key may treat data as anonymous. Adopted for public consultation; final version expected late 2025. **Regulatory authority — primary source for GDPR-compliant pseudonymization.**

[2] National Institute of Standards and Technology (NIST). "SP 800-188: De-Identifying Government Datasets: Techniques and Governance." Final publication, September 14, 2023. https://csrc.nist.gov/pubs/sp/800/188/final — Comprehensive U.S. government guidance on de-identification techniques including suppression, generalization, noise addition, and synthetic data. Covers governance structures (Disclosure Review Boards), risk evaluation frameworks, and the tradeoff between data accuracy and privacy. Notes that differential privacy has matured but is not yet practical for many agencies. **U.S. federal standard — the most detailed technical guidance on de-identification governance.**

[3] Sweeney, Latanya. "k-Anonymity: A Model for Protecting Privacy." International Journal of Uncertainty, Fuzziness and Knowledge-Based Systems, 10(5), 2002, pp. 557–570. Original re-identification experiment: 1997, demonstrating that 87% of Americans were uniquely identifiable by ZIP code + birth date + gender. Published as: https://epic.org/wp-content/uploads/privacy/reidentification/Sweeney_Article.pdf — **Foundational work. Created the field of formal privacy models. Directly shaped HIPAA's Safe Harbor de-identification standard.**

[4] Machanavajjhala, Ashwin, et al. "l-Diversity: Privacy Beyond k-Anonymity." ACM Transactions on Knowledge Discovery from Data (TKDD), 1(1), 2007. Originally presented at ICDE 2006. https://users.cs.duke.edu/~ashwin/pubs/ldiversityTKDDdraft.pdf — Identified the "homogeneity attack" and "background knowledge attack" against k-anonymity. Proposed l-diversity as a stronger guarantee requiring diverse sensitive attribute values within each equivalence group. **Extends Sweeney's work to cover sensitive attribute disclosure, not just identity disclosure.**

[5] Li, Ninghui, Tiancheng Li, and Suresh Venkatasubramanian. "t-Closeness: Privacy Beyond k-Anonymity and l-Diversity." IEEE International Conference on Data Engineering (ICDE), 2007. https://www.cs.purdue.edu/homes/ninghui/papers/t_closeness_icde07.pdf — Demonstrated that l-diversity is insufficient when the distribution of sensitive attributes within a group is skewed relative to the population. Proposed t-closeness using Earth Mover's Distance as the distribution metric. **Completes the k/l/t privacy model hierarchy used by most de-identification frameworks today.**

[6] Apple Differential Privacy Team. "Learning with Privacy at Scale." Apple Machine Learning Journal, 2017. https://machinelearning.apple.com/research/learning-with-privacy-at-scale — Describes Apple's deployment of local differential privacy across hundreds of millions of devices for keyboard analytics, emoji usage, and Safari data. Demonstrates that privacy-preserving analytics are feasible at massive scale. Also: IEEE Digital Privacy, "Differential Privacy and Applications," https://digitalprivacy.ieee.org/publications/topics/differential-privacy-and-applications/ — **Practitioner evidence that differential privacy works in production, not just theory. Relevant as an alternative or complement to pseudonymization.**

[7] ARX Data Anonymization Tool. Open-source software for anonymizing sensitive personal data. Supports k-anonymity, l-diversity, t-closeness, differential privacy, and risk analysis. https://arx.deidentifier.org/ — Also: Google Cloud Sensitive Data Protection provides built-in k-anonymity and l-diversity analysis. **Practical tooling for implementing the privacy models described in this document.**

[8] Databricks. "LogSentinel: How Databricks Uses Databricks for LLM-Powered PII Detection and Governance." Databricks Blog, 2024. https://www.databricks.com/blog/logsentinel-how-databricks-uses-databricks-llm-powered-pii-detection-and-governance — Describes an LLM-powered column classification system achieving 92% precision and 95% recall for PII detection, with automated JIRA ticket creation for remediation. **Practitioner evidence that automated PII scanning works at enterprise scale — supports the ongoing validation approach in Option B.**

[9] ENISA (European Union Agency for Cybersecurity). "Pseudonymisation Techniques and Best Practices." November 2019. https://www.enisa.europa.eu/publications/pseudonymisation-techniques-and-best-practices — Technical guidance on pseudonymization methods including counter-based, random number generators, cryptographic hash functions, HMACs, and encryption. Evaluates each technique against scalability, re-identification risk, and computational cost. Predates the EDPB guidelines but remains the most detailed *technical* reference. **The technical companion to EDPB's governance-focused guidelines.**

[10] Garfinkel, Simson L. "NISTIR 8053: De-Identification of Personal Information." National Institute of Standards and Technology, October 2015. https://nvlpubs.nist.gov/nistpubs/ir/2015/nist.ir.8053.pdf — Earlier NIST publication providing foundational definitions and taxonomy for de-identification. Covers the distinction between anonymization and pseudonymization, direct vs. indirect identifiers, and re-identification risk assessment. **Useful primer and definitional reference — complements the more prescriptive SP 800-188.**
