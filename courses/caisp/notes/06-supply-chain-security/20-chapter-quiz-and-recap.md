# Chapter 6 Quiz and Recap: AI Supply Chain Security

Course: CAISP (Practical DevSecOps)
Scope: 19 question chapter quiz, each answer explained, distractors ruled out, and course wording tightened or flagged where it is loose or dated.

## How to read this document

Each entry gives the question, the correct answer, a short reason it is correct, a one line note on why the other options are wrong, and a flag where the course answer needs qualifying. The running themes are gathered at the end.

---

## Q1. The correct sequence of an LLM's lifecycle

**Answer: Data Collection → Model Training → Fine Tuning → Model Evaluation → Model Deployment.**

Why correct: each stage depends on the one before. You cannot train without data, cannot fine tune without a base model, and should not deploy without evaluating. For security this order also maps the attack surface in sequence: poisoning enters at data collection and training, a backdoor can be planted during training or fine tuning, evaluation is where you might (or might not) catch it, and deployment is where excessive agency and runtime attacks begin.

Why the others are wrong: every alternative puts deployment, training, or evaluation before its prerequisites (deploying before training, evaluating before training, collecting data after training).

Note: in practice evaluation is iterative and recurs throughout, not only once before deployment, but the linear ordering is the intended answer and is correct as a lifecycle backbone.

---

## Q2. Which is NOT a vector for data poisoning

**Answer: Model inversion.**

Why correct: data poisoning means corrupting the data a model learns from. That can happen in foundational model training, in fine tuning, and in Retrieval Augmented Generation (RAG), where an attacker plants malicious documents in the retrieval corpus so they are pulled into the prompt at query time. Model inversion is the opposite direction: it extracts or reconstructs information from an already trained model. It is an attack on confidentiality, not an input corruption, so it is not a poisoning vector.

Why the others are wrong: foundational training, fine tuning, and RAG are all genuine poisoning entry points.

Connection: RAG poisoning is the indirect injection idea from the summariser and RAG labs, seen from the supply chain side. The poisoned document is a supply chain input.

---

## Q3. Model inversion versus model extraction

**Answer: Model inversion attempts to recreate training data, while model extraction creates a copycat model.**

Why correct: the two attacks target different assets. Inversion aims at the private training data (reconstructing faces, records, or text the model memorised). Extraction aims at the model itself, querying it enough to train a functional clone, as in the Carlini et al. "stealing part of a production language model" work from the Chapter 3 notes.

Why the others are wrong: they invent distinctions (small versus large models, physical versus remote access, which loses which) that are not what separates the two.

---

## Q4. An open source LLM that sends data to an external server on initialization

**Answer: Model with a backdoor.**

Why correct: code that runs and phones home at load time is malicious behaviour embedded in the artifact itself. This is exactly the trojanised model labs: a payload that executes when the model is loaded or initialised, establishing contact with attacker infrastructure or exfiltrating data.

Why the others are wrong: model inversion extracts data from a model (it does not make the model act), data poisoning corrupts training inputs (a training time attack, not a load time one), and typosquatting is about a deceptive package or repo name, not runtime behaviour.

Note: at a finer grain this is often delivered through malicious serialisation (the pickle or Lambda layer payloads from the trojan labs). "Backdoor" is the correct umbrella term for the behaviour; malicious serialisation is one common delivery mechanism for it.

---

## Q5. The layer historically affected by Spectre and Meltdown

**Answer: Hardware and cloud platform layer.**

Why correct: Spectre and Meltdown (disclosed 2018) are CPU level flaws in speculative execution. They let one process or tenant read memory it should not, which is especially dangerous in multi tenant cloud where tenants share physical hardware. They sit beneath everything else an AI system runs on.

Why the others are wrong: network, software frameworks, and API layers are higher in the stack; these particular vulnerabilities are in the silicon.

---

## Q6. The sign MOST indicative that an npm package might be malicious

**Answer: The package was published very recently.**

Why correct of the options given: a brand new package is consistent with an attacker standing one up quickly to typosquat or exploit a fresh opportunity. The other three options are not indicators at all: many versions, a complex name, and using TypeScript say nothing about intent.

Flag: recency alone is a weak signal, and plenty of legitimate packages are new. In real triage the stronger red flags are a name that closely mimics a popular package (typosquatting), the presence of install time scripts (`postinstall` hooks), network calls or obfuscated code in the source, a sudden maintainer change, or a version that exists on a public registry but matches an internal package name (dependency confusion). The quiz answer is the best of a weak set, not a complete detection rule.

---

## Q7. The primary purpose of dependency pinning

**Answer: To specify exact versions of dependencies for consistency and security.**

Why correct: pinning locks each dependency to an exact version so builds are reproducible and a later, possibly compromised, release cannot be pulled in silently. It is the same principle as pinning a model's `revision` in the Chapter 2 chatbot labs: reproducibility plus supply chain protection in one move.

Why the others are wrong: pinning reduces flexibility and prevents automatic updates (the opposite of two options), and it does not reduce the number of dependencies.

Note: pinning should be paired with a review and update process, otherwise pinned versions quietly rot and miss security patches. Pin for control, then update deliberately.

---

## Q8. The SLSA level that focuses on provenance by automating the build

**Answer: SLSA Level 1.**

Why correct: in the SLSA Build track, Level 1 requires that the build runs on a platform that automatically generates provenance describing how the artifact was built (what built it, what process, what the top level inputs were). That record lets producers and consumers debug, rebuild, and spot release mistakes such as building from a commit that is not in the upstream repo. Level 2 adds signed provenance from a hosted build platform to deter tampering after the build. Level 3 adds a hardened build platform to deter tampering during the build.

Why the others are wrong: L2 and L3 add protections beyond basic provenance generation, and (see flag) there is no L4 in the current Build track.

Flag: the quiz lists a Level 4 as an option, which reflects the old SLSA v0.1 draft (four levels). The current standard, SLSA v1.0 (and v1.1), restructured into tracks, and the Build track has only L1, L2, and L3 (plus L0 for no guarantees). There is no Build Level 4 today. If you meet "SLSA 4" in older material, it is pre 1.0.

---

## Q9. The MOST effective way to mitigate dependency confusion

**Answer: Creating public packages with the same names as internal packages.**

Why correct of the options given: dependency confusion (Alex Birsan, 2021) works when an attacker publishes a public package whose name matches one of your private internal packages, and the package manager pulls the public one because it has a higher version or is resolved first. Reserving those names yourself in the public registry denies the attacker the name. It is the only defensive option among the four.

Why the others are wrong: closed source only does not address naming, avoiding internal package managers is impractical and unrelated, and changing internal names frequently just creates churn without closing the gap.

Flag: in practice, reserving names publicly is a stopgap, not the strongest primary defence. The stronger controls are scoped namespaces (so internal packages live under an owned scope that cannot be claimed publicly) and configuring the package manager to resolve internal names only from the internal registry, never the public one. The quiz answer is the best of the offered set; the production answer is namespace scoping plus registry resolution rules.

---

## Q10. Container SBOM and Host (VM) SBOM serve identical purposes and contents

**Answer: False.**

Why correct: a container SBOM inventories what is inside a container image (the application, its libraries, runtime components). A host SBOM inventories a whole machine (operating system, system libraries, every installed application). Different scope, different contents, different uses. A container is usually minimal; a host is broad.

---

## Q11. What distinguishes an attestation

**Answer: It is signed, verifiable information making specific claims.**

Why correct: an attestation is a cryptographically signed statement that asserts something specific about an artifact (how it was built, what it contains, which policies it meets) and can be independently verified. The signature makes it tamper evident; the claim makes it meaningful.

Why the others are wrong: an attestation is not limited to SBOM content, not limited to origin tracking, and not exclusive to vulnerability management. Those are too narrow.

---

## Q12. The artifact MOST helpful to check if a disclosed vulnerability affects you

**Answer: Software Bill of Materials (SBOM).**

Why correct: an SBOM is the component and version inventory, so when a CVE lands against a specific library version you search your SBOMs and immediately know which systems contain it. This is exactly the Grype SCA workflow from the Chapter 6 lab: generate the SBOM, match it against a vulnerability database.

Why the others are wrong: provenance tells you how something was built, signatures tell you it is authentic, and attestations without SBOM data make claims, but none of them give the direct component to vulnerability mapping an SBOM does.

---

## Q13. Provenance primarily means securing network traffic between components

**Answer: False.**

Why correct: provenance is the origin and build history of an artifact (its source, build environment, tools, inputs, and the steps that produced it). It establishes trust in the artifact itself, not in the communication channels between running components. Network security is a different concern.

---

## Q14. The combination giving the MOST complete supply chain picture

**Answer: SBOM, provenance data, and attestations.**

Why correct: these three answer different questions and together cover the whole. SBOM answers what is in it, provenance answers how and where it was built, and attestations provide signed, verifiable claims about both. One without the others leaves a gap.

Why the others are wrong: each alternative drops at least one of the three (SBOM plus signatures only, provenance plus scans only, attestations alone).

---

## Q15. How SBOMs, provenance, and attestations relate

**Answer: Attestations can include both SBOM and provenance information.**

Why correct: an attestation is a signed container for claims, and those claims can be an SBOM, or provenance, or both. In the in toto and SLSA model, the attestation is the signed envelope and the SBOM or provenance is the payload (the predicate) inside it. So an attestation unifies component inventory and origin into one verifiable, signed record.

Why the others are wrong: they are not independent with no link, the nesting is not "SBOM contains provenance contains attestations," and provenance is not the sole verifier of the other two.

---

## Q16. The primary difference between signing and attestation

**Answer: Signing only verifies authenticity and integrity, while attestation contains specific claims and metadata.**

Why correct: signing proves an artifact came from the holder of a key and has not changed, and nothing more. An attestation builds on signing by attaching verifiable claims (how it was built, what it contains, which policies it meets), so a consumer can reason about trust beyond "it is unaltered and from this key." This is the exact point from the Cosign labs: a signature answers "authentic and unchanged?" while an attestation answers "and here is what is true about it."

Why the others are wrong: both techniques use cryptography and apply to traditional and AI software alike; the difference is the richness of the signed statement, not keys versus certificates or the domain of use.

---

## Q17. Model Cards and MLBOMs serve identical purposes and contents

**Answer: False.**

Why correct: a Model Card documents model behaviour (performance, intended use, limitations, ethical considerations), aimed at transparency and responsible use. An MLBOM (Machine Learning Bill of Materials) documents the technical supply chain (datasets, parameters, dependencies, provenance), aimed at component tracking and security. Overlapping audiences, different purposes.

---

## Q18. The correct sequence of the model signing process

**Answer: Generating keys → Sharing keys → Signing models → Verifying models.**

Why correct: the producer generates a key pair and protects the private key, then publishes the public key through a trusted channel so consumers can establish trust in the signer before any signed artifact arrives. The producer signs with the private key and distributes models with their signatures, and consumers verify against the public key they already trust.

Why the others are wrong: they verify before signing, sign before generating keys, or (the tempting one) sign and then share the key.

Note, and this is the key lesson: sharing the public key before (or at the same time as) the artifacts is what makes verification meaningful. Publishing the key only after signed models are already circulating is weak, because a consumer then cannot tell the real signer from an attacker who hands over a model, a signature, and a matching key all together. This is precisely the ephemeral key trap from the Cosign in GitLab lab: a key that arrives bundled with the artifact proves consistency, not identity. Trust comes from already holding the right public key, not from the green checkmark.

---

## Q19. Which component would NOT typically be in an MLBOM

**Answer: User feedback ratings.**

Why correct: an MLBOM tracks the technical supply chain of the model, so training dataset details, model parameters, and model provenance all belong. User feedback ratings are operational and evaluation data about the deployed model's reception, not a component or dependency used to build it, so they are out of scope.

Why the others are wrong: dataset details, parameters, and provenance are all core MLBOM contents.

---

## Themes to carry forward

1. **The lifecycle is the attack surface in order.** Data to training to fine tuning to evaluation to deployment. Poisoning enters early, backdoors mid build, and runtime attacks at the end. Knowing the order tells you where each control belongs.

2. **The three artifacts answer three different questions.** SBOM: what is in it. Provenance: how and where it was built. Attestation: a signed claim that can carry either or both. You need all three for a complete picture, and the attestation is the signed envelope the others ride in.

3. **Signing is not the same as a claim, and not the same as safety.** Signing proves authentic and unchanged. An attestation adds verifiable claims. Neither proves the contents are benign, which is why signing pairs with scanning (the Grype, Bandit, ModelScan, Picklescan labs) and why verification only means something against a key or identity you already trust (the Cosign labs and Q18).

4. **Supply chain defences are about naming and versioning discipline.** Pin exact versions, reserve or scope your package names against dependency confusion, and treat a very new package that mimics a known one with suspicion. These are mundane and highly effective.

5. **Attacks target either the data, the model, or the artifact.** Poisoning corrupts the data, inversion and extraction steal from the model, and backdoors live in the artifact. Matching an attack to its target is how most of these questions are answered quickly.

6. **Mind the version of the standard.** SLSA is now v1.0 and v1.1 with a Build track of L1 to L3; the old four level scheme is pre 1.0. CycloneDX added the ML BOM in 1.5. Quoting the current structure matters when this knowledge goes into an audit or interview.

## Sources

1. SLSA security levels (current Build track L1 to L3): https://slsa.dev/spec/v1.0/levels
2. What is new in SLSA v1.0 (tracks and levels restructuring): https://slsa.dev/spec/v1.0/whats-new
3. Alex Birsan, dependency confusion research (2021): https://medium.com/@alex.birsan/dependency-confusion-4a5d60fec610
4. in toto attestation framework (attestation as signed envelope for SBOM or provenance predicates): https://github.com/in-toto/attestation
