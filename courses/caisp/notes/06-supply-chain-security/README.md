# Chapter 6: Supply Chain Attacks in AI

Securing the AI supply chain: understanding the attacks, building a vetting process, and proving artifact integrity through SBOMs, provenance, attestation and signing.

| File | Lesson | Covers |
| ---- | ------ | ------ |
| [01-overview-of-supply-chain-security.md](01-overview-of-supply-chain-security.md) | An Overview of Supply Chain Security | The numbers, NIST lifecycle stages, SolarWinds, Log4Shell, Codecov, CCleaner |
| [02-introduction-to-ai-supply-chain-attacks.md](02-introduction-to-ai-supply-chain-attacks.md) | Introduction to AI Supply Chain Attacks | AI stacking on software and infrastructure, lifecycle stages, why autonomy changes the consequences |
| [03-data-model-and-infrastructure-attacks.md](03-data-model-and-infrastructure-attacks.md) | Data, Model, and Infrastructure Based Attacks | Four column taxonomy, poisoning entry points, inversion vs extraction, PoisonGPT, backdoors |
| [04-abusing-generative-ai-for-package-masquerading.md](04-abusing-generative-ai-for-package-masquerading.md) | Abusing Generative AI for Package Masquerading | Hallucinated packages, slopsquatting, reading registry metadata |
| [05-creating-a-vetting-process.md](05-creating-a-vetting-process.md) | Creating a Vetting Process | Ten step framework, applied to AI components |
| [06-automation-of-vetting-and-third-party-code.md](06-automation-of-vetting-and-third-party-code.md) | Automation of Vetting and Third-Party Code | SAST, SCA, DAST, config management, CI/CD as enforcement, limits of automation |
| [07-scanning-for-vulnerabilities.md](07-scanning-for-vulnerabilities.md) | Scanning for Vulnerabilities | Layer by layer scanning, AI frameworks carry ordinary CVEs, model scanning with modelscan and garak |
| [08-mitigating-dependency-confusion.md](08-mitigating-dependency-confusion.md) | Mitigating Dependency Confusion | Name resolution ambiguity, five mitigations, the `confused` tool |
| [09-dependency-pinning.md](09-dependency-pinning.md) | Dependency Pinning | Exact versions across five ecosystems plus model revisions, lockfiles, the update trade off |
| [10-slsa.md](10-slsa.md) | SLSA | Provenance, Build track L0 to L3, SLSA vs SCVS |
| [11-software-component-verification-standard.md](11-software-component-verification-standard.md) | Software Component Verification Standard | Six control families, three levels, pedigree vs provenance |
| [12-generate-a-software-bill-of-materials.md](12-generate-a-software-bill-of-materials.md) | Generate a Software Bill of Materials | What an SBOM is, benefits, CycloneDX and SPDX |
| [13-application-sbom.md](13-application-sbom.md) | Application SBOM | Scope, fields, generation with Syft and Trivy |
| [14-container-sbom.md](14-container-sbom.md) | Container SBOM | Image contents, base OS, how models ship |
| [15-hosts-sbom.md](15-hosts-sbom.md) | Hosts SBOM | VM scope, OS, runtimes, containers |
| [16-sboms-provenance-and-attestations.md](16-sboms-provenance-and-attestations.md) | SBOMs, Provenance, and Attestations | Three distinct questions, in-toto statements, signing vs attestation |
| [17-model-cards-and-mlboms.md](17-model-cards-and-mlboms.md) | Model Cards and MLBOMs | Cards for humans, ML-BOM for machines |
| [18-model-signing.md](18-model-signing.md) | Model Signing | Authenticity and integrity, Sigstore, signed is not safe |
| [19-chapter-summary.md](19-chapter-summary.md) | Summary | Three movements, corrections, distinctions, five ideas |

Exercises for this chapter live in [../../exercises/](../../exercises/).
