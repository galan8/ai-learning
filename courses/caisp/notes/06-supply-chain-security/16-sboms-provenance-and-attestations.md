# SBOMs, Provenance, and Attestations

## 1. Three different questions

These three are constantly confused. They answer distinct questions, and the value comes from having all three.

| Artifact | Question it answers |
| -------- | ------------------- |
| **SBOM** | **What is in it?** The inventory of components |
| **Provenance** | **Where did it come from and how was it made?** Origin and build history |
| **Attestation** | **Who says so, and can I verify that claim?** A signed statement about the artifact |

```
   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
   │    SBOM      │   │  PROVENANCE  │   │ ATTESTATION  │
   │              │   │              │   │              │
   │  contents    │   │  origin and  │   │  signed      │
   │  inventory   │   │  build       │   │  claim about │
   │              │   │  history     │   │  either      │
   └──────────────┘   └──────────────┘   └──────────────┘
         │                   │                   │
         └───────────────────┴───────────────────┘
                             │
                    all three needed:
         an unsigned SBOM is a claim nobody stands behind
```

## 2. Attestation

An **attestation** is **signed information making specific, verifiable claims** about an artifact.

Structurally an attestation contains:

1. **A subject**: the artifact being described, identified by hash
2. **A predicate**: the claim being made about it
3. **A signature**: from the identity making the claim

The predicate is what varies. In practice attestations carry:

- **Provenance data**: this was built from this repository at this commit, by this build system
- **SBOM data**: these are the components in it
- **Security scan results**: this passed these checks on this date
- **Code review evidence**: this change was reviewed and approved by these people
- **Test results**, policy compliance, vulnerability scan output

The standard format is **in-toto attestations**, which is what SLSA provenance is expressed in and what Sigstore signs.

## 3. Signing versus attestation

The course notes make this distinction well and it is worth stating precisely, because it is the conceptual heart of the lesson.

| | **Signing** | **Attestation** |
| --- | ----------- | --------------- |
| **Goal** | Prove **who signed** something | Prove **who said what** about something |
| **Provides** | Authenticity and integrity | Authenticity, integrity **and specific claims** |
| **Claim content** | None. A signature asserts nothing beyond "this identity signed this bytes" | Structured metadata making verifiable statements |
| **Example** | This file was signed by Jane Doe | Jane Doe states that this artifact was built from repository X at commit Y, contains these components, and passed these scans |

**The practical consequence:** a signature tells you the artifact has not been altered since someone signed it. It does not tell you that the signer built it correctly, reviewed it, scanned it, or that it contains what you think. Attestation is signing **plus the statement worth verifying**.

## 4. How they compose

A mature pipeline emits several attestations for one artifact:

```
   BUILD produces:  myapp:1.4.2  (digest sha256:abc...)
                          │
        ┌─────────────────┼─────────────────┐
        ▼                 ▼                 ▼
   ┌──────────┐    ┌──────────┐     ┌──────────┐
   │ SLSA     │    │ SBOM     │     │ Scan     │
   │provenance│    │attestation│    │ results  │
   │attestation│   │          │     │attestation│
   └──────────┘    └──────────┘     └──────────┘
        │                 │                 │
        └─────────────────┼─────────────────┘
                          ▼
              Policy engine at deploy time:
              "only admit artifacts with valid
               provenance from our build system,
               an SBOM, and a clean scan"
```

That last box is the payoff. Attestations are only useful if something **verifies** them and refuses to proceed when they are missing or invalid. Admission controllers and policy engines (Kyverno, OPA/Gatekeeper, Sigstore policy-controller) are where attestations turn from documentation into enforcement.

## 5. Sigstore

The tooling that made this practical. **Sigstore** provides:

- **Cosign**: signs and attaches signatures and attestations to container images and other artifacts
- **Fulcio**: issues short lived certificates bound to an OIDC identity, so **no long lived private keys need managing**
- **Rekor**: a public transparency log recording that a signature was made, so signing events cannot be quietly retracted

Keyless signing is the reason adoption accelerated: the hardest part of signing was always key management, and Fulcio removes it by binding signatures to workload identity instead.

```bash
# Sign a container image (keyless, via OIDC identity)
cosign sign myorg/llm-inference:1.4.2

# Attach an SBOM as an attestation
cosign attest --predicate app-sbom.json --type cyclonedx myorg/llm-inference:1.4.2

# Verify both
cosign verify myorg/llm-inference:1.4.2
cosign verify-attestation --type cyclonedx myorg/llm-inference:1.4.2
```

## 6. Why this matters for AI

The AI supply chain has weaker provenance than conventional software, and every incident in the earlier chapters exploited that gap.

- **PoisonGPT** succeeded because a model on a public hub carried no verifiable claim about its origin. An attestation binding the weights to a build and a publisher would have made the typosquatted upload detectable.
- **The Hugging Face token exposure** allowed models in real repositories to be replaced. Attestation shifts trust from "the repository said so" to "the publisher's key said so".
- **Mithril's stated motivation for PoisonGPT was precisely this traceability gap.**

## 7. Summary

1. SBOM says what is in it, provenance says where it came from, attestation says who claims it and proves the claim
2. An attestation is a subject, a predicate and a signature, usually in in-toto format
3. Signing proves authorship and integrity only; attestation adds verifiable claims
4. Multiple attestations attach to one artifact: provenance, SBOM, scan results, reviews
5. Sigstore (cosign, Fulcio, Rekor) made this practical by removing key management
6. Attestations become controls only when a policy engine verifies them and refuses non compliant artifacts
