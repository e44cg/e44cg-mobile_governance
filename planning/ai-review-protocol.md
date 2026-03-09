# AI-Powered Multi-Layer Review Protocol

**Status:** Draft | **Date:** March 9, 2026 | **Author:** Claude (for Emilio's review)
**Companion to:** `secure-data-pseudonymization.md`

---

## What this document is

A reusable protocol for using multiple independent AI reviewers + automated testing to validate code, configurations, and transformations — as a rigorous alternative to traditional single-human code review.

The pseudonymization ETL (stripping PII from datasets) is the primary use case, but this protocol applies to any scenario where:
- The code is **structural** (configuration-like, rule-based, checklist-verifiable) rather than algorithmically complex
- The person responsible for approving the work **cannot read code**
- The consequences of a bug are **significant but bounded** (data leakage, misconfiguration, access control errors)
- The verification question can be stated as a **clear, checkable property** ("Does PII survive in the output?" / "Does this config grant access to the right roles?" / "Does this pipeline send data only to approved destinations?")

---

## Part 1: The core protocol

### Why multi-layer AI review works

The academic evidence on AI code review is nuanced and important to understand.

**What AI does well:** A 2024 study at Google on their AutoCommenter tool found that AI-assisted code review achieves high accuracy for structural, rule-based checks — identifying coding standard violations, missing patterns, and checklist-style verification. At a confidence threshold of 0.98, roughly 80% of predictions below the threshold were still correct, indicating the tools are conservative rather than reckless [1].

**What AI does poorly:** A 2025 IEEE-ISTAS paper by researchers demonstrated a "feedback loop security degradation" phenomenon — when code is iteratively refined through AI assistance alone, new vulnerabilities frequently emerge even when the AI is explicitly asked to improve security [2]. This finding is critical: it means using a *single* AI session to write and then self-review code is unreliable. The protocol below addresses this directly by using *independent* AI sessions for authoring and review.

**The multi-agent advantage:** A comprehensive survey published in ACM Transactions on Software Engineering and Methodology (July 2025) found that multi-agent LLM systems — where specialized agents independently review and cross-validate each other's work — outperform single-agent approaches for verification tasks. The collaborative mechanisms (debate, independent review, cross-validation) were "instrumental in encouraging divergent thinking, enhancing factuality and reasoning, and ensuring thorough validation" [3]. The GPTLens framework for smart contract vulnerability detection exemplifies this: independent auditor agents identify issues, then a critic agent filters false positives — a direct parallel to this protocol's structure [3].

**The practitioner reality:** PwC's 2025 guidance on validating multi-agent AI systems advocates a "layered, modular testing and validation framework" where both individual agents and the system as a whole are validated independently [4]. Their key insight: validation must address risk "at both the agent level and the system level."

### The five-step protocol

| Step | Who does it | What they do | Why it matters |
|------|------------|-------------|----------------|
| **1. Author** | AI #1 | Writes the code/configuration based on a clear specification (e.g., PII field inventory, access control requirements, data flow rules) | The starting point — AI drafts the artifact |
| **2. Independent review** | AI #2 (separate session, no context from step 1) | Reviews the artifact against the specification with a single clear question: "Does this artifact satisfy the stated property?" | Avoids the "feedback loop degradation" identified by the IEEE-ISTAS study [2]. Fresh context, no anchoring bias. |
| **3. Automated test** | Automated script | Generates test inputs with known properties (e.g., fake PII), runs the artifact, scans output for violations using pattern matching, schema validation, or property-based testing | Mechanical verification — catches issues that both AI reviewers might miss |
| **4. Test result review** | AI #3 | Reviews the test results: Did the tests catch everything? Are there edge cases? Any violations the automated checks didn't flag? | Third independent perspective — covers blind spots in the test design itself |
| **5. Human approval** | Responsible person | Reviews a plain-language summary: what was the specification, what was built, what was tested, what was found. Stamps approval on the *process and results*, not on individual code lines. | The human validates the outcome and the protocol, not the code |

### Why independence between steps matters

The IEEE-ISTAS finding [2] cannot be overstated: a single AI session that writes code and then reviews its own work is *less reliable* than separate, independent sessions. This is not because AI is bad at review — it's because it anchors on its own prior decisions. The same cognitive bias exists in human review (authors reviewing their own work), but it's more pronounced in AI because the model's context window creates a strong recency bias.

**Rule: AI #2 must not see AI #1's reasoning.** It receives only the artifact (the code/configuration) and the specification (what it should do). It does not see the prompt that generated the code, the conversation that led to it, or any explanation of design choices. This forces a genuinely independent assessment.

This mirrors the cross-validation technique described by Widyasari et al., where "multiple LLMs' answers are validated against each other" — a pattern the ACM TOSEM survey identifies as a key mechanism for improving factuality [3].

### What the human summary looks like

The summary presented at step 5 should read something like:

> **Review Summary — [Project Name / Artifact Type]**
>
> **Specification:** [What the artifact was supposed to do — in plain language]
>
> **What was built:** [Brief description of the artifact]
>
> **What was verified:** [The property being checked]
>
> **Automated test results:** [Number of test cases, what was scanned for, results]
>
> **Independent review findings:** [Any flags from AI #2 or AI #3]
>
> **Recommendation:** Approve / Approve with conditions / Do not approve
>
> **Flags:** [Any concerns, edge cases, or items requiring follow-up]

The responsible person reads this, confirms the specification matches their understanding, and approves. They don't need to read code.

---

## Part 2: Where this protocol applies

### Use case 1: Pseudonymization ETL (primary)

**The scenario:** SQL code strips PII from a dataset and replaces it with pseudonymous IDs. This is the critical trust boundary in the pseudonymization architecture described in `secure-data-pseudonymization.md`.

**Why it's well-suited:** The SQL is structural — column selection, ID replacement, optional generalization (dates → months, zip codes → prefixes). The verification question is clear: "Does any PII column survive into the output?" This is a checklist task, and AI is excellent at checklist tasks.

**Step 3 specifics (automated testing):**
- Generate synthetic data with known PII patterns using tools like the SPY Dataset methodology (Savkin et al., NAACL 2025) [5] or purpose-built generators with realistic PII formats (names, emails, phones, SSNs, addresses)
- Run the SQL against the synthetic data
- Scan the output using regex-based pattern matching for PII formats (email patterns, phone number formats, SSN formats, common name lists)
- Compare output columns against an approved field whitelist
- Run k-anonymity checks on quasi-identifier combinations (k ≥ 5)

**Key design principle — default-deny field selection:**

The ETL SQL should be written as a **whitelist**, not a blacklist:

- **Correct (whitelist):** `SELECT transaction_date, amount, category, region FROM source` — only explicitly named fields pass through. New columns are automatically excluded.
- **Wrong (blacklist):** `SELECT * EXCEPT (name, email, phone) FROM source` — new columns pass through by default. A new PII column added to the source would leak.

This single design decision prevents the most common source of PII leakage: schema changes that add new fields. NIST SP 800-188 recommends this approach, noting that "organizations should not assume that removing direct identifiers is sufficient — the process must account for changes in the data over time" [6].

**Example human summary:**

> **ETL Review Summary — [Project Name]**
>
> **PII fields in source data:** Name, Email, Phone, Date of Birth, Account Number
>
> **Fields stripped and replaced with IDs:** All five PII fields removed from output, replaced with pseudonymous UUID
>
> **Fields remaining in pseudonymized output:** Transaction Date, Transaction Amount, Product Category, Region (3-digit zip), Age Range (bucketed)
>
> **Quasi-identifier assessment:** Region + Age Range + Product Category combination — checked, no unique combinations in dataset (k-anonymity ≥ 5)
>
> **Synthetic test results:** 10,000 fake records processed. Output scanned for email patterns, phone patterns, SSN patterns, name matches. **Zero PII detected in output.**
>
> **Flags:** None.
>
> **Recommendation:** Approve for use with real data.

### Use case 2: N8N / automation workflow configuration

**The scenario:** An N8N workflow (or Zapier, Make, Airflow, etc.) is configured to process pseudonymized data, call AI APIs, and output results. The verification question: "Does this workflow send data only to approved destinations? Does it have access only to the pseudonymized data store, not the PII vault?"

**Why it's well-suited:** Workflow configurations are declarative — they define data flows, API endpoints, credentials, and routing rules. The review is essentially "follow the data": trace every path from input to output and verify that no path leads to an unauthorized destination and no path ingests from an unauthorized source.

**Step 3 specifics (automated testing):**
- Export the workflow configuration as JSON
- Parse the JSON to extract all API endpoints, connection strings, and credential references
- Verify that no endpoint points to the PII vault or raw data source
- Verify that no credential has access to restricted resources
- Run the workflow against synthetic data and capture all outbound network requests — verify they go only to approved destinations

**When it applies:** Any time an automation workflow handles data that has privacy implications, whether pseudonymized datasets, client communications, or analytics pipelines.

### Use case 3: Access control and permissions configuration

**The scenario:** Database roles, dashboard permissions, API access policies, or cloud IAM rules are configured to enforce the pseudonymization architecture's access boundaries. The verification question: "Does this configuration grant access only to the intended resources for the intended roles?"

**Why it's well-suited:** Access control is inherently structural — it's a matrix of who can do what to which resources. The review checks each entry against a specification. Mistakes are high-impact (wrong permissions can expose PII) but the verification is mechanical.

**Step 3 specifics (automated testing):**
- For database roles: run test queries as each role and verify that restricted tables/columns return "access denied"
- For dashboard permissions: verify that each role sees only the expected data views
- For API policies: make test API calls with each credential and verify the response scope
- For IAM: use cloud provider policy simulators (AWS IAM Policy Simulator, GCP Policy Troubleshooter) to verify effective permissions

**When it applies:** Any time access controls are created or modified in a system that handles sensitive data. Particularly important during initial setup and after any role/permission changes.

### Use case 4: Dashboard and report templates

**The scenario:** A BI dashboard or report template is designed to display pseudonymized analysis results. The re-identification drill-down is available only to approved roles. The verification question: "Does this dashboard show PII to any unapproved role? Does the re-identification feature require proper authorization?"

**Why it's well-suited:** Dashboard configurations are declarative — data source connections, filter logic, role-based visibility rules, drill-down definitions. The review traces what each role can see and verifies that PII is only accessible through the controlled re-identification path.

**Step 3 specifics (automated testing):**
- Log in as each role and capture screenshots/exports of every view
- Verify that analyst roles see only pseudonymous IDs
- Verify that re-identification requires explicit action (click-through, authentication challenge) and is logged
- Export the dashboard's data source connections and verify they point only to the pseudonymized data store (not the PII vault) for standard views

### Use case 5: Data pipeline schema validation

**The scenario:** A data pipeline's schema evolves — new columns are added to the source, transformations are modified, downstream consumers change. The verification question: "Has any schema change introduced a PII leakage path?"

**Why it's well-suited:** Schema changes are the most common source of PII leakage in production systems. A new column added to the source data that isn't handled by the ETL whitelist could flow through to the pseudonymized store. This is a recurring validation task, not a one-time review.

**Step 3 specifics (automated testing):**
- Compare the current source schema to the approved schema — flag any new columns
- Verify that new columns are either explicitly included in the whitelist (with justification) or automatically excluded
- Run the PII pattern scanner against the pseudonymized data store after any schema change
- This can be automated as a CI/CD step or a scheduled job — Databricks' LogSentinel system demonstrates this approach, achieving 92% precision and 95% recall for automated PII detection with JIRA ticket creation for any violations [7]

**When it applies:** Every time the source data schema changes, on a recurring schedule for ongoing pipelines, and as a post-deployment validation step.

---

## Part 3: When this protocol is NOT sufficient

### Regulated industries with explicit human-code-review requirements

Some compliance frameworks require a qualified human to review the actual code:
- **HIPAA Security Rule** (45 CFR §164.312): Requires "technical security measures to guard against unauthorized access to electronic protected health information." While it doesn't explicitly mandate human code review, audit and certification processes often interpret this as requiring qualified human oversight of security-critical code.
- **SOC 2 Type II**: Requires evidence that controls are operating effectively over time. Auditors may require human-reviewed code as evidence, though the trend is toward accepting automated testing evidence.
- **PCI DSS v4.0**: Requirement 6.3.2 mandates code review for custom software — either manual or automated, but the process must be documented and the reviewer must be qualified.

In these cases, the AI review protocol still adds value as a *first pass* — it catches obvious issues before the human reviewer spends time on it. The human review becomes the final step rather than the only step.

### Extremely sensitive data

For data with life-safety or national-security implications (protected health information in clinical contexts, classified government data, financial data subject to fiduciary obligations), add a qualified third-party reviewer. The protocol reduces their workload by pre-filtering issues, but doesn't replace them.

### Client contracts that specify review procedures

Some clients require specific review processes in their data processing agreements. Check the contract before relying on AI-only review. The protocol can usually satisfy or exceed contractual requirements, but this must be verified per engagement.

### Novel or complex algorithmic code

This protocol works best for **structural** code — transformations, configurations, access rules, schema definitions. For complex algorithmic code (ML model training, statistical analysis, custom business logic), the verification question is harder to state as a clear, checkable property. In those cases, use this protocol as one layer within a broader quality assurance process, not as the sole review mechanism.

The IEEE-ISTAS study specifically found that AI security degradation is worst in iterative, complex code refinement [2]. For simple structural code (the sweet spot of this protocol), the risk is much lower because the verification is checklist-based rather than reasoning-based.

---

## Part 4: Risks and mitigations

### Risk: Quasi-identifier re-identification

Even after stripping all explicit PII, combinations of remaining fields can sometimes uniquely identify a person. Sweeney's 1997 demonstration — re-identifying the Governor of Massachusetts using ZIP code, birth date, and gender from a $20 voter registration database — remains the canonical example [8]. Her subsequent research established that 87% of Americans are uniquely identifiable from just these three quasi-identifiers.

**Mitigation:**
- Generalize quasi-identifiers (exact dates → months, exact amounts → ranges, full zip → 3-digit prefix)
- Run k-anonymity checks: for every combination of quasi-identifiers, are there at least k records that share that combination? k=5 is a common threshold [8]
- For datasets with sensitive attributes, apply l-diversity (Machanavajjhala et al., 2006) to ensure diversity of sensitive values within each equivalence group [9]
- For datasets where distributional skew is a concern, apply t-closeness (Li et al., 2007) using Earth Mover's Distance [10]
- Include quasi-identifier assessment in the review protocol (steps 2 and 4)
- Use the open-source ARX tool for automated privacy model verification [11]

### Risk: PII vault as high-value target

The mapping table (ID → real identity) is the crown jewel. If it's breached, all pseudonymization is undone.

**Mitigation:**
- Store in a separate database/schema from the pseudonymized data — different credentials, different access controls. The EDPB guidelines require that re-identification keys be stored separately with independent technical and organizational measures [12].
- Encrypt at rest
- Network-level isolation where possible (the AI tools should have no network path to the vault)
- Audit-log every read
- Regularly review who has access; remove access that's no longer needed

### Risk: ETL code bugs / schema drift

A transformation error or an unhandled schema change could pass PII through to the pseudonymized store. This is the risk the entire review protocol is designed to catch.

**Mitigation:**
- The five-step protocol above
- Default-deny (whitelist) field selection — new columns are excluded by default
- Automated PII scanning as a recurring check (not just at initial setup): periodically scan the pseudonymized data store for PII patterns. SPY (Savkin et al., NAACL 2025) demonstrates that synthetic PII datasets trained specifically for PII detection outperform generic NER models for this task [5].
- Any schema change to the source data triggers a re-run of the review protocol
- Automated alerting when the pseudonymized data store contains columns not in the approved field list

### Risk: AI-generated code introducing subtle vulnerabilities

A 2025 empirical analysis of 7,703 AI-generated code files on GitHub found that while 87.9% did not contain identifiable CWE-mapped vulnerabilities, Python code exhibited vulnerability rates of 16–18%, and the types of vulnerabilities shifted over time toward deeper architectural flaws (privilege escalation paths up 322%) [13]. For simple structural SQL (the primary use case of this protocol), the risk is lower than these aggregate numbers suggest — the code is not complex enough for most architectural vulnerability classes. But the finding reinforces the importance of steps 2–4 (independent review and automated testing).

**Mitigation:**
- The independence requirement between AI #1 (author) and AI #2 (reviewer) directly addresses the feedback loop degradation identified by the IEEE-ISTAS study [2]
- Automated testing (step 3) catches issues regardless of whether they were introduced by AI or human
- For high-stakes engagements, use different AI models for authoring and review (e.g., Claude for authoring, GPT for review) to reduce the chance of shared blind spots

### Risk: Scope creep on dashboard re-identification

If too many people get drill-down access to resolve IDs to real clients, the pseudonymization becomes meaningless in practice.

**Mitigation:**
- Explicit approval process for re-identification access (not just "anyone with a login")
- Time-limited access where possible (access expires after the analysis session)
- Audit trail on every re-identification lookup — the EDPB guidelines require that any re-identification attempt be detectable [12]
- Periodic review of who has access and whether they still need it

---

## Part 5: Summary

The multi-layer AI review protocol (author → independent review → automated test → test result review → human approval) provides rigorous validation for structural code and configurations without requiring the responsible person to read code. The human approves the *process and outcomes*, not the artifact itself.

The protocol is strongest for:
- **ETL pseudonymization code** (primary use case)
- **Automation workflow configurations** (data routing, API connections)
- **Access control and permissions** (who can see what)
- **Dashboard templates** (what each role sees)
- **Schema validation** (ongoing monitoring for PII leakage)

It is weakest for complex algorithmic code, where the verification question cannot be stated as a simple checkable property.

The key design decision is **independence**: the AI that reviews must not see the context of the AI that authored. This directly addresses the empirical finding that iterative AI self-review degrades security rather than improving it [2].

---

## Sources

[1] Vijayvergiya, Manushree, et al. "AI-Assisted Assessment of Coding Practices in Modern Code Review." Proceedings of the 1st ACM International Conference on AI-Powered Software (AIware '24), July 2024. https://homes.cs.washington.edu/~rjust/publ/code_review_automation_aiware_2024.pdf — Google's study on AutoCommenter, an AI-assisted code review tool. Found high accuracy for structural, rule-based checks at a 0.98 confidence threshold, with roughly 80% of sub-threshold predictions still correct. **Empirical evidence from the largest-scale deployment of AI code review — supports the claim that AI excels at checklist-style structural verification.**

[2] IEEE-ISTAS 2025. "Security Degradation in Iterative AI Code Generation: A Systematic Analysis of the Paradox." https://arxiv.org/html/2506.11022v2 — Controlled experiment (40 rounds across 4 prompting strategies) demonstrating that code refined through automated AI assistance alone frequently develops new vulnerabilities, even when explicitly asked to improve security. **Critical finding that directly motivates the independence requirement in this protocol — a single AI session should not author and review its own work.**

[3] Rasheed, Zeeshan, et al. "LLM-Based Multi-Agent Systems for Software Engineering: Literature Review, Vision and the Road Ahead." ACM Transactions on Software Engineering and Methodology (TOSEM), July 2025. https://dl.acm.org/doi/10.1145/3712003 — Comprehensive survey of multi-agent LLM systems in software engineering. Covers collaborative mechanisms (debate, independent review, cross-validation) and their effectiveness. Describes GPTLens, a framework using independent auditor agents + critic agent for vulnerability detection. **Peer-reviewed survey confirming that multi-agent cross-validation improves factuality and thoroughness — the theoretical foundation for this protocol's multi-step structure.**

[4] PwC. "Validating Multi-Agent AI Systems: From Modular Testing to System-Level Governance." 2025. https://www.pwc.com/us/en/services/audit-assurance/library/validating-multi-agent-ai-systems.html — Industry guidance advocating layered validation for multi-agent systems, addressing risk at both the individual agent level and the system level. Draws on system-theoretic approaches (STAMP). **Practitioner perspective from a Big Four firm — provides credibility for the protocol's layered review approach in client-facing contexts.**

[5] Savkin, Vladislav, et al. "SPY: Enhancing Privacy with Synthetic PII Detection Dataset." Proceedings of NAACL 2025, Student Research Workshop. https://aclanthology.org/2025.naacl-srw.23/ — Introduces a synthetic dataset for PII detection using LLM-generated data that emulates real-world PII scenarios. Found that dedicated PII detection models outperform generic NER models for privacy-specific tasks. **Academic evidence supporting the use of synthetic PII data for testing — directly applicable to Step 3 of the protocol.**

[6] National Institute of Standards and Technology (NIST). "SP 800-188: De-Identifying Government Datasets: Techniques and Governance." Final publication, September 14, 2023. https://csrc.nist.gov/pubs/sp/800/188/final — Recommends that de-identification processes account for changes in data over time and advocates for ongoing monitoring rather than one-time review. **Federal standard supporting the schema validation use case (Use Case 5) and the default-deny principle.**

[7] Databricks. "LogSentinel: How Databricks Uses Databricks for LLM-Powered PII Detection and Governance." Databricks Blog, 2024. https://www.databricks.com/blog/logsentinel-how-databricks-uses-databricks-llm-powered-pii-detection-and-governance — LLM-powered PII classification system achieving 92% precision and 95% recall, with automated JIRA ticket creation for violations. **Practitioner evidence that automated PII scanning works at enterprise scale — supports Use Case 5 (schema validation) and Step 3 (automated testing).**

[8] Sweeney, Latanya. "k-Anonymity: A Model for Protecting Privacy." International Journal of Uncertainty, Fuzziness and Knowledge-Based Systems, 10(5), 2002, pp. 557–570. https://epic.org/wp-content/uploads/privacy/reidentification/Sweeney_Article.pdf — **Foundational work establishing quasi-identifier re-identification risk and the k-anonymity model.**

[9] Machanavajjhala, Ashwin, et al. "l-Diversity: Privacy Beyond k-Anonymity." ACM Transactions on Knowledge Discovery from Data (TKDD), 1(1), 2007. https://users.cs.duke.edu/~ashwin/pubs/ldiversityTKDDdraft.pdf — **Extends k-anonymity to protect against homogeneity and background knowledge attacks on sensitive attributes.**

[10] Li, Ninghui, Tiancheng Li, and Suresh Venkatasubramanian. "t-Closeness: Privacy Beyond k-Anonymity and l-Diversity." IEEE ICDE, 2007. https://www.cs.purdue.edu/homes/ninghui/papers/t_closeness_icde07.pdf — **Addresses distributional privacy using Earth Mover's Distance; completes the k/l/t hierarchy.**

[11] ARX Data Anonymization Tool. Open-source software supporting k-anonymity, l-diversity, t-closeness, differential privacy, and risk analysis. https://arx.deidentifier.org/ — **Practical tooling for implementing privacy models.**

[12] European Data Protection Board (EDPB). "Guidelines 01/2025 on Pseudonymisation." January 16, 2025. https://www.edpb.europa.eu/system/files/2025-01/edpb_guidelines_202501_pseudonymisation_en.pdf — **Primary regulatory guidance for GDPR-compliant pseudonymization.**

[13] Security Vulnerabilities in AI-Generated Code: A Large-Scale Analysis of Public GitHub Repositories. October 2025. https://arxiv.org/abs/2510.26103 — Empirical analysis of 7,703 AI-generated code files. Found 4,241 CWE instances across 77 vulnerability types. Python showed highest vulnerability rates (16–18%). **Large-scale empirical evidence on AI code vulnerability rates — contextualizes the risk this protocol mitigates.**

[14] Apiiro. "4x Velocity, 10x Vulnerabilities: AI Coding Assistants Are Shipping More Risks." June 2025. https://apiiro.com/blog/4x-velocity-10x-vulnerabilities-ai-coding-assistants-are-shipping-more-risks/ — Industry analysis finding 10,000+ new security findings per month from AI-generated code by June 2025, with privilege escalation paths up 322% and architectural design flaws up 153%. **Practitioner data on the scale of AI code security risks — reinforces the need for independent review and automated testing.**
