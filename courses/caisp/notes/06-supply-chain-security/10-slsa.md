# SLSA

## 1. What it is

**SLSA** (Supply chain Levels for Software Artifacts, pronounced "salsa") is a security framework maintained by the **OpenSSF**, originally from Google. It is a checklist of standards and controls that answers one question:

> **Where did this artifact come from, and can I prove it?**

SLSA is about **build integrity**. It does not tell you whether your dependencies have vulnerabilities. It tells you whether the thing you are about to run was actually produced from the source you think it was, by the build system you think built it, without tampering in between.

## 2. Provenance, the core concept

**Provenance is the origin and history of an artifact**: where a component came from and how it was made. SLSA formalises provenance as **signed, machine readable metadata** emitted by the build system, recording:

1. **What** was built (the output artifact and its hash)
2. **From what source** (repository and commit)
3. **Who or what** built it (the build platform identity)
4. **How** it was built (the build process and parameters)
5. **When** it was built

Provenance helps in three ways, which is the framing worth keeping:

| Benefit | What it gives you |
| ------- | ----------------- |
| **Tracking** | Trace any artifact back to its source and build |
| **Authenticity** | Confirm it came from who it claims, cryptographically |
| **Transparency** | Make the build process inspectable rather than assumed |

## 3. The levels

**Version note.** The course material likely teaches SLSA v1.0 (2023). The current specification is **v1.2, approved November 2025**, which is backwards compatible with v1.0 and v1.1 Build track claims. The significant change is that **v1.2 promotes the Source track from experimental to approved**, so SLSA now has two tracks rather than one.

### Build track (levels 0 to 3)

| Level | Name | What it requires |
| ----- | ---- | ---------------- |
| **Build L0** | None | No guarantees. The default state |
| **Build L1** | Provenance exists | The build produces provenance describing what was built, how and by whom. Enables manual detection of mistakes. Transparency, not tamper resistance |
| **Build L2** | Hosted build platform | Builds run on a hosted platform that **signs and generates** the provenance itself. The signature means provenance cannot be trivially forged by the developer |
| **Build L3** | Hardened builds | The build platform provides **strong tamper resistance**: isolated build environments, non falsifiable provenance, secrets inaccessible to the build |

The progression is worth understanding as a single idea: **L1 says the build described itself, L2 says the platform vouched for that description, L3 says the platform is hard enough to attack that the vouching means something.**

### Source track

Added because build integrity is pointless if the source was tampered with first. It defines levels covering version control, history integrity, enforced technical controls, and two party review of changes.

## 4. How you actually get it

You mostly do not implement SLSA by hand. Modern platforms emit it:

1. **GitHub Artifact Attestations** provide **Build Level 2 by default**, and Level 3 with reusable workflows
2. **npm trusted publishing** from GitHub Actions or GitLab CI automatically generates and publishes provenance
3. **slsa-github-generator** provides builders for Go, Node.js, Maven, Gradle and containers
4. Verification is done with `slsa-verifier` against the provenance file

Provenance is typically signed via **Sigstore**, which is the connective tissue between SLSA (what to claim) and the signing infrastructure (how to prove it).

## 5. Why this matters for AI

Everything in the AI supply chain is a build artifact: container images serving models, the framework packages, the fine tuning pipeline outputs, and increasingly the models themselves.

The **torchtriton dependency confusion attack** from the LLM Top 10 notes is the clearest case. A malicious package was pulled because the package manager resolved a name to the wrong source. Provenance would not have prevented publication of the malicious package, but it would have let a consumer verify that the `torchtriton` they received came from the PyTorch project's build system rather than an anonymous PyPI account.

**The general principle: SLSA turns "I downloaded this from the right URL" into "I can cryptographically verify this came from the right build."** Those are very different assurances, and only the second survives an attacker who controls a registry, a mirror, or a name.

## 6. Summary

1. SLSA answers where an artifact came from and whether that can be proven
2. Provenance is signed metadata about what, from where, by whom, how and when
3. Build track L0 to L3: none, provenance exists, platform signed, hardened and tamper resistant
4. Current version is v1.2 (November 2025), which added an approved Source track
5. You get it from your platform (GitHub Artifact Attestations, npm trusted publishing) rather than building it yourself
6. It complements rather than replaces vulnerability scanning: SLSA proves origin, scanning finds flaws
