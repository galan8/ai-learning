# Chapter 6 Summary

## The argument in one line

You inherit the security of everything upstream of you, the AI layer stacks on top of the software layer without removing any of its risk, and the defence is a chain of controls that makes the supply chain visible, verifiable and deliberate.

## The chapter in three movements

```
   UNDERSTAND THE PROBLEM        BUILD THE PROCESS         PROVE AND VERIFY
   (lessons 1 to 4)              (lessons 5 to 9)          (lessons 10 to 18)
   ─────────────────────         ─────────────────         ────────────────
   Supply chain security         Vetting framework         SLSA (provenance)
   AI stacks on software         Automation in CI/CD       SCVS (maturity)
   Data, model, infra,           Scanning every layer      SBOMs (what is in it)
     development attacks         Dependency confusion      Provenance (how made)
   AI hallucinated packages      Dependency pinning        Attestation (who says)
                                                           Model cards + ML-BOM
                                                           Model signing
```

The movements are sequential for a reason. You cannot build a vetting process without knowing what you are vetting against, and you cannot verify anything without first knowing what you have.

## Lesson by lesson

| # | Lesson | The point |
| - | ------ | --------- |
| 1 | Overview of supply chain security | Three quarters of your code is someone else's; 742% average annual attack growth; six of seven vulnerabilities are transitive |
| 2 | Introduction to AI supply chain attacks | AI stacks on software and infrastructure, inheriting all their risk and adding autonomy |
| 3 | Data, model, and infrastructure attacks | Four columns: data, model, infrastructure, development. Inversion steals data, extraction steals the model |
| 4 | Package masquerading | Models invent package names, attackers register them; repeatable hallucinations make it scale |
| 5 | Creating a vetting process | Ten steps from requirements to documentation; a framework, not a tool |
| 6 | Automation of vetting | Manual does not scale; CI/CD is where automation becomes enforcement |
| 7 | Scanning for vulnerabilities | Every layer needs its own scanner; AI stacks carry ordinary CVEs |
| 8 | Mitigating dependency confusion | A name resolution ambiguity, not a vulnerability; claim your namespace |
| 9 | Dependency pinning | Exact versions make every change deliberate; pair with an update process |
| 10 | SLSA | Signed provenance; Build track L0 to L3 |
| 11 | SCVS | Six control families, three levels; assesses your verification maturity |
| 12 | Generate an SBOM | The inventory; CycloneDX and SPDX |
| 13 | Application SBOM | What one application was built from |
| 14 | Container SBOM | Everything in the image, including base OS. How models ship |
| 15 | Hosts SBOM | Everything on the machine |
| 16 | Provenance and attestations | SBOM says what, provenance says how, attestation says who claims it |
| 17 | Model cards and ML-BOMs | Cards for humans, ML-BOMs for machines |
| 18 | Model signing | Authenticity and integrity for artifacts nobody can read |

## Corrections to the course material and to my notes

| Position in the notes | Correct position |
| --------------------- | ---------------- |
| 743% average annual increase in supply chain attacks | **742%**, from Sonatype's State of the Software Supply Chain research, measured since 2019 |
| 1.3 billion vulnerable dependencies downloaded monthly | Sonatype reports **1.2 billion** avoidable known vulnerable downloads monthly in one framing and **3.4 billion** monthly downloads of vulnerable software with a fix available in another. Quote either with its framing, not a figure between them |
| Iranian actors exploited "Log5j" | **Log4j / Log4Shell**, exploited by Iranian government sponsored actors against a US federal agency via unpatched VMware Horizon (CISA advisory) |
| CycloneDX 1.6 introduced ML-BOM | **ML-BOM arrived in CycloneDX 1.5 (June 2023)**; 1.6 added attestations and a cryptography BOM; 1.7 shipped October 2025 (ECMA-424 second edition) |
| SLSA and SCVS "have some resemblance" | Related but distinct. **SLSA proves how an artifact was built**; **SCVS assesses how well you verify what you consume** |
| SLSA levels unspecified | **Build track L0 to L3, cumulative.** The v0.1 four level model with hermetic builds at L4 was retired in 2023 |
| "Kokro" TTS model | **Kokoro-82M** (hexgrad): text to speech, 82 million parameters, ONNX, Apache 2.0 |
| "sextuple code" | **sample code**, a standard model card section |
| Signing and attestation used interchangeably | **Signing proves who signed something** (authenticity and integrity, no claims). **Attestation proves who said what** (specific verifiable claims plus metadata) |
| "No high profile AI infrastructure supply chain breaches" | Accurate as stated, and worth keeping. That column is largely theoretical exposure rather than documented incident, unlike the data and model columns |

## The distinctions worth memorising

**Model inversion versus model extraction.** Inversion reconstructs the **training data**; extraction reconstructs the **model**. Both work through ordinary API queries, which is why query rate limits and restricting returned confidence scores defend against both.

**Provenance versus pedigree.** Provenance is where a component came from. **Pedigree is whether it was modified since**: forked, patched, recompiled, fine tuned. Clean provenance plus undisclosed pedigree is exactly where a backdoor lives, and it is the AI ecosystem's weakest point.

**Model card versus ML-BOM.** The card is **for humans** (capabilities, intended use, limitations, biases). The ML-BOM is **for machines** (a structured inventory you can scan, diff and query). You read the card to decide whether to use a model; you scan the ML-BOM to find out whether you are affected.

**Signing versus attestation.** "X signed this file" versus "X signed this file, along with this SBOM, built from this repository at this commit."

## What the chapter does not cover, and should

1. **VEX (Vulnerability Exploitability eXchange).** An SBOM says a vulnerable component is present; VEX says whether it is actually exploitable in your context. Without it, SBOM driven vulnerability management buries teams in findings that do not apply.
2. **SBOM drift.** An SBOM is accurate only at the moment of generation. Regenerating per build and diffing between builds is what makes it useful rather than ceremonial.
3. **Dataset provenance as a governance problem.** ML-BOM has fields for it; whether a dataset was lawfully collected is not something a format can answer.
4. **Enforcement.** Every artifact in this chapter is voluntary. The tooling to generate and verify all of it exists today, and supply chain attacks keep working because consumers do not check.
5. **Agentic supply chain.** MCP servers, tools and plugins are now suppliers in the chain, with their own poisoning, rug pull and cross tenant failure modes.

## The five ideas worth keeping

1. **Most exposure is known, already fixed, and transitive.** Six of seven vulnerabilities come through dependencies you never chose, and 96% of vulnerable downloads had a fixed version available. This is not primarily a zero day problem.
2. **The distribution channel is the weapon.** SolarWinds, CCleaner and Codecov all arrived through the victim's own trusted update or build process. "Download from the official source" is not a defence.
3. **Transparency is a prerequisite, not a control.** An SBOM prevents nothing; it makes "are we affected?" answerable in minutes rather than weeks.
4. **Signed is not safe.** A validly signed pickle based model still executes code on load. Authenticity, integrity and safety are three different properties.
5. **AI adds a layer and removes nothing.** The novel risks (poisoned models, hallucinated packages) sit on top of every conventional risk, and in practice the CVE that gets you is still in `pillow` or `setuptools`.
