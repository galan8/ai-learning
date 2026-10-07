# Software Component Verification Standard (SCVS)

## 1. What it is

**OWASP SCVS** (note the letter order: **S**oftware **C**omponent **V**erification **S**tandard) is a community driven framework of activities, controls and best practices for **identifying and reducing risk in a software supply chain**.

Where SLSA asks "can you prove how this was built?", SCVS asks a broader question: **"does your organisation have a disciplined process for knowing, verifying and managing what goes into your software?"**

## 2. SLSA and SCVS: related, not the same

The course notes that they resemble each other. They are complementary, and the distinction is worth being precise about:

| | SLSA | SCVS |
| --- | ---- | ---- |
| **Focus** | Build integrity and provenance of an artifact | Organisational process for verifying components |
| **Question** | Where did this come from and can I prove it? | Do we have the controls to manage what we consume? |
| **Scope** | The build pipeline | Inventory, SBOM, build environment, package management, analysis, pedigree |
| **Unit** | An artifact | A programme or organisation |
| **Output** | A signed provenance attestation | A maturity level across control families |

Use SLSA to prove an artifact's origin. Use SCVS to assess whether your supply chain practice is mature, and to sequence improvement.

## 3. Three verification levels

Higher levels include all controls from the levels below.

| Level | For | Characteristic |
| ----- | --- | -------------- |
| **Level 1** | Low assurance requirements, where basic analysis suffices | Groundwork: complete and accurate SBOMs, repeatable builds via continuous integration, analysis of third party components using publicly available tools and intelligence. **Achievable with modern software engineering practice** |
| **Level 2** | Moderately sensitive software needing additional due diligence | Builds on L1 and brings in additional stakeholders beyond engineering, including contracts and procurement. Assumes some risk management maturity |
| **Level 3** | High assurance requirements due to data sensitivity or safety | Critical infrastructure and safety systems. Requires **auditability and end to end transparency** across the supply chain |

**A design point worth noting:** because SCVS is tiered *and* topical, an organisation can sit at different levels in different control families. You are not required to reach Level 2 everywhere before adopting a Level 3 control that matters to you. OWASP explicitly recommends tailoring.

## 4. The six control families

1. **Inventory.** Do you know what components you use, everywhere?
2. **Software Bill of Materials.** Are SBOMs structured, machine readable, timestamped, uniquely identified, complete, accurate, and analysed for risk? Higher levels add signature existence and signature *verification*.
3. **Build Environment.** Are builds repeatable, isolated, and free from unnecessary access to secrets and networks? This is the family that overlaps most directly with SLSA.
4. **Package Management.** Can the repository correlate published versions back to source in version control? Is code signing required? Higher levels require MFA for publishers and SCA performed before publication.
5. **Component Analysis.** The process of identifying risk from open source and third party components: known vulnerabilities, licensing, maintenance status.
6. **Pedigree and Provenance.** Is the origin of a component known and verifiable? If a component was **modified**, is that modification documented, uniquely identified, and analysed with the same rigour as an unmodified one?

Family 6 is the one that matters most for AI, and the least commonly practised. A fine tuned model is a modified component, and a LoRA adapter on a poisoned base is a modified component whose pedigree is broken.

## 5. What sits inside these controls

The course notes group several practices under SCVS; here they are correctly attributed.

**Security testing** contributes to Component Analysis:

- **SAST** (static analysis of source)
- **DAST** (dynamic analysis of a running application)
- **SCA** (software composition analysis: identifying third party components and their known issues)
- **Fuzz testing** (malformed input to find crashes and memory safety problems)

**Vulnerability management** is a three step cycle:

1. **Identification.** Use tooling and vulnerability databases to find known issues. **Terminology correction:** you find **CVEs** (the identifiers for specific vulnerabilities); **CVSS** is the scoring system that rates their severity. The course notes conflate the two.
2. **Assessment.** Determine exploitability and relevance in your context. A CVE in a code path you never execute is not the same risk as one on your ingress.
3. **Remediation.** Patch, upgrade, replace, or accept with compensating controls.

**Compliance with industry standards** means adherence to frameworks such as NIST, OWASP and ISO, maintaining records, and updating them regularly. In the AI context add the EU AI Act, NIST AI RMF and ISO/IEC 42001.

**Risk management** wraps the whole thing: evaluate risks, implement mitigations, monitor continuously, run security audits and assessments, establish and update policies, and make it a collaboration across teams rather than a security team activity.

## 6. Continuous, not one time

SCVS emphasises controls that can be **implemented or verified through automation**, for a specific reason: software composition changes constantly in a CI pipeline, so **continuity of assurance requires continuous verification rather than a point in time audit**. An SBOM generated once at release and never regenerated is a snapshot of a system that no longer exists.

## 7. Summary

1. SCVS is a framework for assessing and improving supply chain verification practice
2. Three levels: basic hygiene, additional due diligence, full auditability for critical systems
3. Six control families: inventory, SBOM, build environment, package management, component analysis, pedigree and provenance
4. Complementary to SLSA: SLSA proves an artifact's build, SCVS assesses your process
5. Levels can differ per family; tailoring is expected
6. Verification must be continuous, because composition changes continuously
