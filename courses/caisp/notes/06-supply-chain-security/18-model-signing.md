# Model Signing

## 1. The problem it solves

Models can be edited and poisoned. PoisonGPT proved a single fact could be surgically altered and the result uploaded under a typosquatted name, with benchmarks differing by 0.1%. JFrog found around a hundred malicious models on Hugging Face, one of which opened a reverse shell on load. Meta's LLaMA weights leaked and circulated as torrents.

In every one of those cases the consumer had **no way to answer two questions**:

1. **Authenticity:** did this model actually come from the publisher it claims to come from?
2. **Integrity:** is it byte for byte what that publisher released, or has it been modified since?

Model signing answers both cryptographically, and establishes a **chain of custody** from publisher to production.

**Why it matters more for models than for most artifacts:** you cannot read a model. You can review source code, diff a config, inspect a container layer. You cannot look at several gigabytes of floating point weights and tell whether a backdoor is present. Signing is not a convenience here; for the consumer it is very nearly the only available evidence.

## 2. The mechanism

Standard public key cryptography applied to model files.

```
   PUBLISHER SIDE                          CONSUMER SIDE
   ──────────────                          ─────────────

   1. Generate a key pair
      ┌─────────────┐  ┌─────────────┐
      │ private key │  │ public key  │──────────┐
      └──────┬──────┘  └─────────────┘          │
             │                                  │
   2. Sign each model file                      │  3. Publish the
      with the private key                      │     public key
             │                                  │
             ▼                                  ▼
      ┌────────────────┐                 4. Verify signature
      │ model.safeten- │ ── distributed ──►    against the
      │ sors + .sig    │                       public key
      └────────────────┘                          │
                                                  ▼
                                       ✔ authentic and untampered
                                       ✘ wrong publisher or modified
```

1. **Generate keys.** The publisher creates a private and public key pair.
2. **Sign.** Each model file is signed with the private key, producing a signature over the file's cryptographic digest.
3. **Share the public key**, making it accessible for verification.
4. **Verify.** Consumers check the signature against the public key before loading the model. A valid signature proves it came from the holder of the private key and has not been altered.

**Sign every file, not just the weights.** A model repository contains config files, the tokenizer, and frequently custom Python that runs on load. Signing `model.safetensors` while leaving `modeling_custom.py` unsigned protects the part you cannot execute and leaves unprotected the part you can.

## 3. Signing versus attestation, applied to models

From the previous lesson, and the distinction matters here:

| | Signing a model | Attesting a model |
| - | --------------- | ----------------- |
| Claim | "This organisation released this file" | "This model was trained from this dataset, using this code, on this platform, and passed these evaluations" |
| Verifies | Authenticity and integrity | Specific, verifiable statements about origin and process |
| Detects | Tampering, impersonation, typosquatting | Everything above, plus provenance of training |

A signature would have stopped PoisonGPT's distribution, because the typosquatted organisation could not produce a valid signature from the real one's key. **An attestation would additionally have exposed the ROME edit**, because the attested provenance would not match a legitimate training run.

Signing is the floor. Attestation is where the ecosystem is heading.

## 4. Keys, and the problem with them

Traditional signing has a key management problem: private keys must be protected, public keys must be distributed in a way consumers can trust, and rotation is painful. If the publisher's private key leaks, an attacker signs poisoned models that verify perfectly.

**Sigstore** is the modern answer and is worth knowing for the exam and for practice. It provides **keyless signing**: rather than managing long lived keys, the signer authenticates with an existing identity (an OIDC provider), receives a short lived certificate from Fulcio, signs, and the signing event is recorded in **Rekor**, a public append only transparency log.

Two properties follow, and they are the reason this approach won:

1. **No long lived private key to steal.** The certificate expires in minutes.
2. **Transparency.** Every signature is publicly logged, so an unexpected signing event against your identity is detectable. You cannot quietly sign a poisoned model.

Sigstore is what signs SLSA provenance in practice, which closes the loop with lesson 10.

## 5. The state of model signing

Worth knowing where this actually stands, because it is moving:

1. **The OpenSSF AI/ML Security Working Group publishes a model signing project** built on Sigstore, aimed specifically at signing model artifacts and their associated files.
2. **Hugging Face supports signed commits and integrates malware scanning and pickle scanning**, but signature verification is not universally enforced by consumers, which is the weak link. A signature nobody checks is decoration.
3. **The practical gap is enforcement, not availability.** The tooling exists. Most pipelines still pull a model by name and load it without verifying anything.

## 6. Practical guidance

1. **Verify before loading, in the pipeline, as a gate.** Verification that happens manually or optionally does not happen.
2. **Combine signing with revision pinning.** A signature proves who published it; a pinned commit hash proves *which version* you are getting. Every lab in this course pinned a `revision_id`, which is the lightweight form of this control.
3. **Prefer safetensors.** Signing tells you a file is authentic; it does not make a pickle file safe to deserialise. A validly signed malicious model is still a malicious model. Authenticity and safety are different properties.
4. **Mirror verified models internally** and have production pull from the mirror, so verification happens once at a controlled boundary rather than implicitly at every deploy.
5. **Record the verification result** as part of your ML-BOM or attestation chain, so an auditor can see not just that a signature exists but that you checked it.

## 7. Summary

1. Model signing gives consumers authenticity and integrity, the two questions they otherwise cannot answer.
2. It matters more for models than most artifacts, because weights cannot be reviewed by inspection.
3. Mechanism: generate keys, sign each file with the private key, publish the public key, verify before use.
4. Sign every file in the repository, including custom code that executes on load.
5. Signing proves who released it; attestation proves how it was produced. Signing is the floor.
6. Sigstore replaces long lived keys with short lived certificates plus a public transparency log.
7. The tooling exists; the gap is that consumers rarely verify. A signature nobody checks is decoration.
8. Signed does not mean safe: prefer safetensors regardless.
