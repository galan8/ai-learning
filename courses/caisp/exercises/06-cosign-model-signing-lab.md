# Exercise: Signing and Verifying Machine Learning Models with Cosign

Course: CAISP (Practical DevSecOps)
Status: Complete (Cosign installed, key pair generated, BERT model signed and verified, verification script built)

## How to read this document

For the concept with no code, read **Part 1** and **Part 5**. To reproduce the lab, work through **Part 3**. **Part 4** carries the one idea that matters most and that the lab does not state outright: what signing does and does not prove.

This is the defensive answer to the attack labs in this chapter. The trojanised model labs showed that a model file is executable code that can be tampered with; ROME and PoisonGPT showed that a model's knowledge can be silently edited and redistributed under a look alike name. Signing is the control that lets a consumer detect both.

---

# Part 1: Introduction (for everyone)

## What we are doing

We take a real machine learning model downloaded from a public hub, and we attach a cryptographic signature to it using a tool called **Cosign**. The signature is like a tamper evident wax seal: anyone who later receives the model can check the seal and know two things for certain, that the file came from the holder of a particular private key, and that not a single byte has changed since it was sealed. Change one byte and the seal breaks.

## The idea in plain terms

Earlier labs showed how easily a model file can be poisoned or booby trapped, and that the corrupted version looks identical to the real one. Signing solves the "looks identical" problem. The publisher runs their private key over the exact bytes of the model to produce a signature. The consumer runs the matching public key over the file they received and the signature; if they match, the file is bit for bit what the publisher signed. If an attacker altered the model anywhere along the way, the check fails.

The important limit, which Part 4 returns to: this proves *who signed it and that it is unchanged*. It does **not** prove the model is safe or truthful. A malicious publisher can perfectly sign a malicious model. Signing establishes authenticity and integrity, not virtue.

## Why it matters

1. **It detects tampering in transit and at rest.** A model swapped or modified after the publisher released it will fail verification.
2. **It ties a model to an identity.** "Verified OK against this key" is only meaningful if you have independently decided to trust that key, which turns the security question into the manageable one of "do I trust this publisher's key?" rather than "is this giant binary safe?".
3. **It creates an audit trail.** Cosign can record each signature in a public, tamper proof log, so there is durable evidence of what was signed and when.

## What to take away

Signing gives model files the same chain of custody that code signing gives software. It is the concrete mechanism behind the "verify provenance" mitigation that appears in every attack lab in this chapter. Combined with scanning (which checks for badness) it is one half of a complete model intake process: scanning asks "is this dangerous?", signing asks "is this genuinely the publisher's unaltered file?".

---

# Part 2: First Principles

**Principle 1: A signature binds a file's exact contents to a private key.**
Signing computes a cryptographic hash (a short fingerprint) of the file, then encrypts that fingerprint with the signer's private key. The result is the signature. Because the fingerprint changes completely if any byte of the file changes, the signature covers the whole file.

**Principle 2: Verification uses the matching public key.**
The public key can check a signature but cannot create one. So the publisher keeps the private key secret and hands out the public key freely. Anyone with the public key can verify, but only the holder of the private key can sign. This asymmetry is the whole point.

**Principle 3: Integrity and authenticity are different from safety.**
A passing verification tells you the file is unchanged (integrity) and came from the key holder (authenticity). It says nothing about whether the contents are benign. This is the seam that ties this lab to the attack labs: a signed trojan is still a trojan.

**Principle 4: Trust flows from the key, not from the check.**
"Verified OK" is only as meaningful as your reason to trust the key that did the verifying. If you verify against a key an attacker generated, of course it passes; it is the attacker's own signature on the attacker's own file. The hard part of any signing system is not the maths, it is deciding which keys or identities are trusted and getting the right public key to the verifier.

**Principle 5: A transparency log makes signatures accountable.**
Cosign is part of **Sigstore**, whose components include **Rekor**, a public append only transparency log. Recording a signature there means it cannot later be quietly denied or altered, which is useful for audit and for detecting a compromised key being misused.

---

# Part 3: Step by Step Replication

## 3.0 Environment

1. Lab environment (Linux) with internet access to GitHub and Hugging Face
2. `wget` for downloading
3. About 500 MB free (the BERT model is 420 MB)

## 3.1 Install Cosign

Cosign ships as a single binary. Download it, move it onto the path, make it executable:

```bash
wget "https://github.com/sigstore/cosign/releases/download/v2.6.1/cosign-linux-amd64"

mv cosign-linux-amd64 /usr/local/bin/cosign
chmod +x /usr/local/bin/cosign

cosign version
```

`mv` places the binary in `/usr/local/bin`, a directory already on the system path so `cosign` works from anywhere. `chmod +x` marks it executable.

A small inconsistency to be aware of: the lab downloads `v2.6.1` but the `cosign version` output in the lab shows `v2.4.1`, which means the lab box already had an older Cosign installed and the `mv` did not overwrite it, or the transcript was captured across two versions. It does not affect the exercise (both versions behave identically here), but if you are checking your work against the printed output, that is why the numbers differ. Verify you are actually running the version you downloaded if it matters to you.

`cosign --help` lists the subcommands. The ones this lab uses are `generate-key-pair`, `sign-blob`, and `verify-blob`. The `-blob` variants are the key detail: most of Cosign is built for container images, but `sign-blob` and `verify-blob` work on any ordinary file, which is what a model is.

## 3.2 Create a project directory

```bash
mkdir model-signing
cd model-signing
```

## 3.3 Generate a key pair

```bash
cosign generate-key-pair
```

It prompts for a password to protect the private key. The lab uses `pdso-admin`. This produces two files:

1. `cosign.key`, the **private** key, encrypted with that password. This signs. It must stay secret.
2. `cosign.pub`, the **public** key. This verifies. It can be shared with anyone.

The lab's own note is the real world lesson: in production the password must be strong and unique, and the private key stored securely (a secrets manager or hardware token), because anyone who obtains the private key and its password can forge signatures that verify as you.

## 3.4 Download a model to sign

```bash
wget https://huggingface.co/bert-base-uncased/resolve/main/pytorch_model.bin
```

This is BERT base uncased, a widely used 420 MB language model, saved as `pytorch_model.bin`. Worth noticing: `.bin` is PyTorch's pickle based format, the exact format the trojan labs weaponised. So this lab is signing precisely the kind of file that can carry a payload, which makes the "signing proves authenticity, not safety" point concrete rather than abstract.

## 3.5 Sign the model

```bash
cosign sign-blob --key cosign.key pytorch_model.bin > model.sig
```

Enter the password (`pdso-admin`) when prompted. Cosign then shows a Sigstore notice about the transparency log and asks for consent, because by default it records the signature in the public Rekor log. Answering `y` prints something like `tlog entry created with index: 198404195`, the model's entry in that public log. The base64 signature is written to `model.sig` (the `>` redirects Cosign's output into that file).

What just happened, from the lab's own summary:

1. Cosign computed a cryptographic hash of `pytorch_model.bin`
2. It signed that hash with your private key
3. It saved the signature to `model.sig`
4. It recorded the signature in the public transparency log (because you consented)

A note on the transparency log consent screen: it warns that the email tied to your signing identity becomes part of an immutable public record. In this lab with a local key that is minimal, but it is a real privacy consideration in keyless signing (below), and worth reading rather than reflexively accepting in real use.

## 3.6 Verify the signature

```bash
cosign verify-blob --key cosign.pub --signature model.sig pytorch_model.bin
```

Output: `Verified OK`.

That confirms three things at once: the signature is valid, the model file matches the one that was signed (unchanged), and the signature was made with the private key matching this public key.

**Prove it works by breaking it.** Append a single byte to the model and verify again:

```bash
echo "x" >> pytorch_model.bin
cosign verify-blob --key cosign.pub --signature model.sig pytorch_model.bin
```

This now fails, because the file's hash no longer matches the signed one. That failure is the entire value of signing: any tampering, however small, is detected. (Re download the model to restore it.)

## 3.7 Wrap verification in a gate

For production, verification belongs in front of model loading, as a gate that refuses to proceed on failure:

```bash
cat > verify_model.sh<<EOF
#!/bin/bash

MODEL_FILE=\$1
SIG_FILE=\$2
KEY_FILE=\$3

if [ ! -f "\$MODEL_FILE" ] || [ ! -f "\$SIG_FILE" ] || [ ! -f "\$KEY_FILE" ]; then
  echo "Usage: \$0 <model_file> <signature_file> <public_key_file>"
  exit 1
fi

echo "Verifying model integrity..."
if cosign verify-blob --key "\$KEY_FILE" --signature "\$SIG_FILE" "\$MODEL_FILE"; then
  echo "Model verified successfully"
  echo "Safe to use this model for inference"
  exit 0
else
  echo "Model verification failed"
  echo "DO NOT use this model, it may have been tampered with"
  exit 1
fi
EOF

chmod +x verify_model.sh
./verify_model.sh pytorch_model.bin model.sig cosign.pub
```

The important part is the exit code. The script returns `0` on success and `1` on failure, so a loading pipeline can make verification a hard precondition: verify first, and load only if the exit code is `0`. A gate that logs a warning but loads anyway is not a gate. (One caveat on the script's own wording: it prints "Safe to use this model", which overstates what happened. Verification proved the file is authentic and unaltered, not that its contents are safe. "Verified authentic" would be the honest message.)

---

# Part 4: Security Analysis

## What signing proves, and what it does not

This is the crux, and the lab's conclusion skips it. Verification answers exactly two questions:

1. **Integrity:** has the file changed since it was signed? (No, if it verifies.)
2. **Authenticity:** was it signed by the holder of the private key behind this public key? (Yes, if it verifies.)

It does **not** answer:

3. **Safety:** are the contents benign? Signing a trojanised model produces a perfectly valid signature. `Verified OK` on a booby trapped pickle just means the booby trap arrived intact from a genuine source.
4. **Trustworthiness of the signer:** verification against a key tells you nothing about whether that key's owner deserves trust. That decision is yours to make, out of band, before you ever verify.

Tie this to PoisonGPT directly. The attacker uploaded a poisoned model under the typosquatted name `EleuterAI` (one letter off `EleutherAI`). If that attacker signs their poisoned model with their own key, it verifies perfectly against their own public key. Signing does not save a victim who fetches the wrong publisher's key. What defeats PoisonGPT is verifying against **the real EleutherAI's known public key or identity**, which fails for the impostor's file. The security lives in trusting the right key, not in the green checkmark.

So signing pairs with, and does not replace, the scanning from the Picklescan, Grype and ModelScan labs:

1. **Scanning** asks: is this file dangerous? (Catches the trojan, misses a swap by a trusted looking source.)
2. **Signing** asks: is this file authentic and unchanged from a specific signer? (Catches the swap and the tamper, misses a genuinely malicious signer.)

You need both. Verify the signature against a trusted publisher identity, *and* scan the verified file before loading.

## Keyed signing (this lab) versus keyless signing (production)

This lab uses **keyed** signing: a long lived private key protected by a password. That works, but it puts the hard problem of key management on you: the private key can be stolen, the password can leak, and you must get the correct public key to every verifier through a trusted channel.

Modern Sigstore practice is **keyless** signing, which removes the long lived key entirely:

1. You authenticate with an identity provider (a Google, GitHub or corporate login via OIDC).
2. Sigstore's **Fulcio** issues a short lived certificate binding a freshly generated key to your verified identity (for example your email or a CI workflow).
3. You sign, the certificate and signature go into the **Rekor** transparency log, and the ephemeral key is discarded.
4. Verifiers check against your **identity** (`--certificate-identity` and `--certificate-oidc-issuer`) rather than a raw public key.

The benefit: nothing long lived to steal, and trust is expressed as "I trust models signed by ci@company.com via our identity provider", which is far more meaningful than "I trust this blob of a public key". This is the direction to note for real deployments, even though the lab teaches the simpler keyed flow first.

## The ML native evolution: OpenSSF Model Signing

Signing one `.bin` file is the teaching case. Real models are directories of many files (config, tokenizer, multiple weight shards). Signing them one blob at a time does not scale and leaves gaps. The **OpenSSF Model Signing (OMS)** project, released as v1.0 in April 2025 by the OpenSSF AI/ML Working Group and built on Sigstore, addresses this: it hashes every file in a model repository into a single manifest and signs the manifest, so one signature covers the entire model and verification detects any added, removed or altered file. It ships as the `model-signing` Python package and integrates with Hugging Face. For production ML provenance this is the current best practice, and this Cosign lab is the conceptual foundation underneath it.

## Where this sits in the course

This lab is the payoff for the whole supply chain thread. Every attack lab ended with "verify provenance" as a mitigation; this is that mitigation in working code. It is the counterpart to the scanning labs (Picklescan, Grype, ModelScan) and the intake companion to the threat model from the Chapter 5 lab, where "unsigned model from an unverified source" is exactly the kind of untrusted input a data flow diagram flags at a trust boundary.

---

# Part 5: Conclusion (for everyone)

We put a tamper evident seal on a machine learning model and then checked it. The seal proves two things and only two: the model came from the holder of a particular private key, and not one byte has changed since they sealed it. We proved the second point by changing a single byte and watching the check fail. That is genuinely valuable, because the attack labs showed how easily a model can be swapped or poisoned into an identical looking fake, and this is how you catch that.

The part to hold onto is what the seal does not promise. It does not promise the model is safe, and it does not promise the person who sealed it deserves your trust. A malicious publisher can seal a malicious model flawlessly, and it will verify perfectly against their key. The famous PoisonGPT attack worked by getting people to trust a publisher name that was one letter wrong; a signature would not have saved them unless they checked it against the real publisher's identity. So the real work is deciding whose key or identity you trust and obtaining the genuine one, and then, separately, still scanning the verified file for danger before you load it.

Signing answers "is this authentic and unchanged?" and scanning answers "is this dangerous?". A serious model intake process asks both, every time, and this lab is the authentic and unchanged half.

---

## Ideas to take forward

1. Experiment: run the tamper test in 3.6 (append a byte, watch verification fail, restore) and keep it as the one line demonstration of why signing works.
2. Experiment: sign the trojanised pickle from the earlier lab, verify it as `Verified OK`, then run ModelScan on that same verified file and watch it flag the payload. This proves in two commands that "verified" and "safe" are different properties.
3. Experiment: try keyless signing with `cosign sign-blob` against an OIDC identity, and verify with `--certificate-identity`, to feel the difference from keyed signing.
4. Experiment: install the `model-signing` package and sign a whole model directory with OMS, comparing the single manifest signature against signing each file by hand.
5. Concept file: `concepts/model-signing.md` on integrity versus authenticity versus safety, keyed versus keyless, Sigstore (Fulcio, Rekor) and OMS, with PoisonGPT as the worked example of why trusting the right identity is the load bearing step.

## Sources

1. Sigstore Cosign documentation (signing and verification overview): https://docs.sigstore.dev/cosign/signing/overview/
2. Cosign source and releases: https://github.com/sigstore/cosign
3. OpenSSF, Launch of Model Signing v1.0 (April 2025): https://openssf.org/blog/2025/04/04/launch-of-model-signing-v1-0-openssf-ai-ml-working-group-secures-the-machine-learning-supply-chain/
4. Sigstore blog, Practical Model Signing with Sigstore: https://blog.sigstore.dev/model-transparency-v1.0/
5. `model-signing` package: https://pypi.org/project/model-signing/
