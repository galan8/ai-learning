# Exercise: Signing an LLM with Cosign in a GitLab CI/CD Pipeline

Course: CAISP (Practical DevSecOps)
Status: Complete (pipeline built and corrected to run reliably; model and metadata signed and verified as pipeline stages)

## How to read this document

For the concept with no code, read **Part 1** and **Part 5**. To reproduce the pipeline, work through **Part 3**, which uses a corrected `.gitlab-ci.yml` that runs cleanly (see the reliability note in 3.5 for what changed and why). **Part 4** covers what signing inside a pipeline does and does not buy you.

This builds directly on the previous lab, `06-cosign-model-signing-lab.md`, which signed a model by hand. Here the same signing and verification run automatically as stages of a CI/CD pipeline, so every model that moves through the pipeline is signed and checked without a person doing it.

---

# Part 1: Introduction (for everyone)

## What we are doing

We put model signing on an assembly line. Instead of a person running Cosign at a keyboard, a GitLab pipeline does it automatically every time: one stage downloads a model, the next signs it, the last verifies the signature and fails the whole pipeline if the check does not pass. The output is a model plus its signature and public key, produced by an automated, repeatable process.

## The idea in plain terms

The previous lab showed the mechanism of signing. On its own, a manual step that a busy engineer has to remember is a step that gets skipped. CI/CD (continuous integration and delivery) is the automation that runs steps for you whenever code changes. By making "sign the model" and "verify the signature" into pipeline stages, signing stops being an optional good habit and becomes a gate that every model has to pass before it can move on. If verification fails, the pipeline goes red and the model does not proceed.

## Why it matters

1. **Automation is what makes security controls actually happen.** A control that depends on memory is unreliable; a control wired into the pipeline runs every time by default.
2. **It produces an audit trail.** Each pipeline run records what was downloaded, signed and verified, with timestamps, which is exactly what compliance and incident response need.
3. **It signs the metadata too, not just the weights.** The pipeline records where the model came from and when, and signs that record, so the claimed provenance is itself tamper evident.

## What to take away

This is the "verify provenance" mitigation from every attack lab in this chapter, turned into automation. It is also where a subtle weakness appears that the hand version hid: when the pipeline generates a throwaway signing key on every run, the signature proves very little to an outside consumer. Part 4 is about that gap and how real deployments close it. The mechanics are the easy part; deciding whose identity signs, and protecting that identity, is the real work.

---

# Part 2: First Principles

**Principle 1: A pipeline is a sequence of stages, each in a fresh container.** GitLab runs `prepare`, then `sign`, then `verify`, each in its own clean container image. Nothing carries between them automatically except files explicitly saved as **artifacts**. That is why Cosign is installed again in each stage that needs it, and why the model and signature are declared as artifacts to pass forward.

**Principle 2: Secrets belong in protected CI variables, not in the code.** The signing password is stored as a GitLab CI/CD variable (`COSIGN_PASSWORD`), read from the environment at run time, so it never appears in the committed `.gitlab-ci.yml`. This is the standard way to keep credentials out of source control.

**Principle 3: The verify stage is a gate because it can fail the pipeline.** Cosign's `verify-blob` returns a non zero exit code when a signature does not match, and a non zero exit code fails the GitLab job, which fails the pipeline. That is what turns verification from a printed message into an enforced control.

**Principle 4: Signing in CI moves the trust boundary to the CI system.** Once the pipeline holds the signing key and defines the steps, whoever can change the pipeline or the runner can change what gets signed. The security of the signature now depends on the security of the pipeline. This is the central idea of Part 4.

**Principle 5: The transparency log is optional but meaningful.** This pipeline disables Sigstore's public log (Rekor) so it does not depend on an external service. That keeps the pipeline reliable and self contained, at the cost of the public, immutable record the log provides. A deliberate tradeoff, not a free choice.

---

# Part 3: Step by Step Replication

## 3.0 Environment

1. The lab's GitLab instance and the DevSecOps box, both provided by the lab
2. GitLab login: user `root`, password `pdso-training` (as given by your lab; the host name in URLs will be your own instance)
3. Cosign, installed on the DevSecOps box for the manual exploration, and installed again inside the pipeline containers

## 3.1 Install Cosign on the box (for exploration)

```bash
wget -O /usr/local/bin/cosign https://github.com/sigstore/cosign/releases/download/v2.6.1/cosign-linux-amd64
chmod +x /usr/local/bin/cosign
cosign --help
```

`cosign --help` lists the subcommands. The pipeline uses `generate-key-pair`, `sign-blob` and `verify-blob`, the same blob (arbitrary file) commands as the previous lab.

## 3.2 Create the GitLab project

In the GitLab web UI, create a new blank project named `cosign_test`, with **root** as the namespace, and leave "Initialize repository with a README" ticked so the repo has a default `main` branch to clone.

## 3.3 Configure Git and clone

On the DevSecOps box (substitute your own instance host name in these commands):

```bash
git config --global user.email "root@<your-gitlab-host>"
git config --global user.name "root"

git clone git@<your-gitlab-host>:root/cosign_test.git
cd cosign_test
```

If prompted to accept the host's SSH fingerprint, type `yes` and press Enter.

## 3.4 Store the signing password as a protected CI variable

In the project, go to **Settings → CI/CD → Variables → Expand**, and add:

| Name | Value |
| ---- | ----- |
| `COSIGN_PASSWORD` | `pdso-admin` |

Cosign reads this from the environment automatically, so the password never appears in the pipeline file. In real use, mark such a variable **Masked** and **Protected**.

## 3.5 Create the pipeline file

Create `.gitlab-ci.yml` with three stages: `prepare` downloads the model, `sign` generates a key and signs the model and its metadata, `verify` checks both signatures.

```bash
cat > .gitlab-ci.yml <<'PIPELINE'
stages:
  - prepare
  - sign
  - verify

variables:
  MODEL_NAME: "tiny-gpt2"
  MODEL_VERSION: "1.0"
  COSIGN_VERSION: "v2.2.3"

prepare_model:
  stage: prepare
  image: python:3.9-slim
  script:
    - apt update && apt install -y wget
    - mkdir -p models
    - wget -O models/pytorch_model.bin https://huggingface.co/sshleifer/tiny-gpt2/resolve/main/pytorch_model.bin
    # fail fast if the download did not return a real file
    - test -s models/pytorch_model.bin || (echo "model download failed or empty" && exit 1)
    - |
      cat > models/model-metadata.json <<META
      {
        "model_name": "${MODEL_NAME}",
        "version": "${MODEL_VERSION}",
        "source": "huggingface/sshleifer/tiny-gpt2",
        "download_date": "$(date -u +"%Y-%m-%dT%H:%M:%SZ")"
      }
      META
  artifacts:
    paths:
      - models/

sign_model:
  stage: sign
  image: python:3.9-slim
  script:
    - apt update && apt install -y wget
    - wget -O cosign https://github.com/sigstore/cosign/releases/download/${COSIGN_VERSION}/cosign-linux-amd64
    - chmod +x cosign
    - mv cosign /usr/local/bin/
    # COSIGN_PASSWORD comes from the protected CI variable
    - cosign generate-key-pair
    - cosign sign-blob --yes --tlog-upload=false --key cosign.key --output-signature models/model.sig models/pytorch_model.bin
    - cosign sign-blob --yes --tlog-upload=false --key cosign.key --output-signature models/metadata.sig models/model-metadata.json
  artifacts:
    paths:
      - models/
      - cosign.pub

verify_model:
  stage: verify
  image: python:3.9-slim
  script:
    - apt update && apt install -y wget
    - wget -O cosign https://github.com/sigstore/cosign/releases/download/${COSIGN_VERSION}/cosign-linux-amd64
    - chmod +x cosign
    - mv cosign /usr/local/bin/
    - cosign verify-blob --key cosign.pub --signature models/model.sig --insecure-ignore-tlog models/pytorch_model.bin
    - cosign verify-blob --key cosign.pub --signature models/metadata.sig --insecure-ignore-tlog models/model-metadata.json
PIPELINE
```

### Reliability note: three changes from the printed lab, so the pipeline runs cleanly

1. **Removed `pip install torch transformers` from the prepare stage.** This is the important one. Those libraries are large (close to a gigabyte with dependencies) and, crucially, are never used: the model is fetched with `wget` and signed with `cosign`, and nothing in the pipeline loads it in Python. On a constrained runner that install is the step most likely to exhaust time, memory or disk and fail the job. Removing it makes `prepare` fast and reliable and changes nothing about the result.
2. **Created the file with a quoted heredoc (`<<'PIPELINE'`) and named the inner heredoc `META`.** Quoting the outer delimiter means `${MODEL_NAME}` and `$(date ...)` are written into the file literally and expanded by the runner at job time (so the metadata date is the run date), rather than being expanded once on the box when you paste. Naming the inner delimiter `META` instead of a second `EOF` removes any chance of it colliding with the outer delimiter.
3. **Added a one line download check** (`test -s`) so that if the model download ever returns an empty file or an error page, the stage fails immediately with a clear message instead of signing a broken file.

Everything else matches the lab: the same three stages, the same `--tlog-upload=false` on signing and `--insecure-ignore-tlog` on verifying, and the same artifacts.

## 3.6 Why the transparency log is disabled here

The signing uses `--tlog-upload=false` and the verifying uses `--insecure-ignore-tlog`, which together switch off Sigstore's public transparency log (Rekor). The lab's own reasoning, which is sound:

1. The public Sigstore instance can be unreliable from some networks, so depending on it can fail the pipeline for reasons unrelated to the model.
2. Skipping it keeps verification entirely self contained, with no external dependency.

The tradeoff, stated honestly by the lab: `--insecure-ignore-tlog` contains "insecure" for a reason. The signatures are still cryptographically checked, but you lose the public, immutable record that the log provides. For production, the guidance is to use the full Sigstore capabilities including the transparency log (or a private Rekor instance).

## 3.7 Push and watch the pipeline

```bash
git add .
git commit -m "add model signing pipeline"
git push origin main
```

Pushing triggers the pipeline. Watch it under the project's **Pipelines** page. On success, all three stages go green, and the **Browse artifacts** view shows:

1. `pytorch_model.bin`, the model
2. `model-metadata.json`, the provenance record
3. `model.sig` and `metadata.sig`, the two signatures
4. `cosign.pub`, the public key a consumer uses to verify

A note that appears in the lab: the runner sets `RUNNER_GENERATE_ARTIFACTS_METADATA` to generate build metadata in **SLSA** format for artifacts built on the platform. That connects this pipeline to the SLSA supply chain framework from the Chapter 6 notes: signing proves the artifact is unchanged, and SLSA provenance describes how and where it was built.

---

# Part 4: Security Analysis

## What automation adds, and the trap it introduces

Automating signing is a real gain: it runs every time, produces an audit trail, and gates progression on a passing check. But this specific pipeline also illustrates the classic weakness of naive CI signing, and it is worth seeing clearly.

**The pipeline generates a brand new key pair on every run.** `cosign generate-key-pair` runs in the `sign` stage, so each pipeline produces a *different* private and public key, uses them once, and the public key rides along in the artifacts. Verification then checks the model against the same public key that was just produced beside it.

That is circular. A consumer who receives the model, the signature and the public key together, and verifies one against the other, has proved only that these three files are mutually consistent. It proves nothing about **who** produced them, because the key is anonymous and freshly minted. An attacker who swaps all three (their model, their signature, their key) passes this verification perfectly. This is the same lesson as the previous lab's PoisonGPT point, sharpened: a signature is only meaningful against a key you independently trust, and a per run throwaway key is a key no one has any prior reason to trust.

So this pipeline is a good demonstration of the *mechanism* and a poor model of *trust*. Real provenance needs one of:

1. **A stable, protected signing identity.** A long lived key kept in a secrets manager or hardware token (or a KMS), whose public key is published once through a trusted channel, so every model the organisation signs verifies against the *same* known key that consumers have pinned.
2. **Keyless signing tied to the pipeline's own identity.** GitLab CI can obtain an OIDC identity token, and Cosign keyless signing (Fulcio plus Rekor) binds the signature to that verified identity, for example "signed by the `root/cosign_test` pipeline on this GitLab instance". Consumers then verify against that identity with `--certificate-identity`, which is a durable thing to trust, rather than an anonymous key. This is the production shape of exactly this pipeline.

## Trust moved into the pipeline

Once signing lives in CI, the trust boundary is the CI system. Whoever can edit `.gitlab-ci.yml`, alter a runner, or read the signing key can forge signatures the organisation will accept. Consequences for how this must be run:

1. Protect the pipeline definition (branch protection, code review on `.gitlab-ci.yml`, protected variables).
2. Protect the signing key or, better, avoid a stored key entirely with keyless signing.
3. Treat the runners as sensitive infrastructure, since a compromised runner can sign anything.

## Signing the metadata, not just the weights

A genuinely good practice in this lab: it signs `model-metadata.json` (source, version, download date) as well as the model file. That means the *claim* about where the model came from is tamper evident too, not just the bytes. A supply chain story where the provenance record is unsigned is weak, because an attacker can rewrite the story; signing it closes that gap. Extending the metadata to include the model's hash and the exact upstream revision would make it stronger still.

## Where this sits in the course

This is the operational form of the model signing lab, and the automation counterpart to the scanning tools (Grype, Bandit, ModelScan, Picklescan) that also belong in a pipeline. A complete CI intake for a model would chain both: verify the signature against a trusted identity, scan the verified file for malicious content, and only then allow it to proceed. Signing plus scanning, enforced as pipeline gates, is the concrete answer to the trojanised model and PoisonGPT attacks from earlier in the chapter.

---

# Part 5: Conclusion (for everyone)

We took model signing off the workbench and put it on an assembly line. A GitLab pipeline now downloads a model, records where it came from, signs both the model and that record, and verifies the signatures, all automatically, failing loudly if anything does not match. That is a real improvement over a manual step, because the control that runs by itself is the control that actually runs.

The lab also, perhaps unintentionally, teaches the limit of naive automation. Because the pipeline mints a fresh, anonymous signing key on every run, its "verified" only means the model and its signature agree with a key that was born moments earlier and that no one has any reason to trust. It is a perfect demonstration of the machinery and a reminder that machinery is not the point. The value of a signature comes entirely from trusting the identity behind it, so a real pipeline signs with a stable, protected identity, or with the pipeline's own verified identity through keyless signing, and consumers verify against that. Trust does not come from the green checkmark; it comes from knowing, and protecting, whose key made it.

Wired up properly, and paired with scanning the verified file, this is how models earn their place in production: authentic, unaltered, from a source you have decided to trust, and checked automatically every single time.

---

## Ideas to take forward

1. Experiment: after a green pipeline, tamper with the model artifact and re run just the verify logic locally against the published `cosign.pub`, to watch verification fail. Confirms the gate works.
2. Experiment: replace the per run key with keyless signing using GitLab's OIDC token, and verify with `--certificate-identity` and `--certificate-oidc-issuer`, to turn the anonymous key into a trustworthy identity.
3. Experiment: add a fourth stage that runs ModelScan on the verified model, so the pipeline enforces both "authentic" and "not malicious" before the model is allowed through.
4. Experiment: extend `model-metadata.json` to include the model's SHA256 and the exact Hugging Face revision, and sign that, so the provenance record pins the precise upstream artifact.
5. Concept file: `concepts/signing-in-cicd.md` on why in pipeline signing moves the trust boundary to the CI system, the ephemeral key trap, keyless signing, and the signing plus scanning intake gate.

## Sources

1. Sigstore Cosign documentation: https://docs.sigstore.dev/cosign/signing/overview/
2. Cosign keyless signing (Fulcio and Rekor): https://docs.sigstore.dev/cosign/signing/overview/
3. GitLab CI/CD variables and OIDC identity tokens: https://docs.gitlab.com/ee/ci/secrets/id_token_authentication.html
4. SLSA supply chain framework: https://slsa.dev/
5. OpenSSF Model Signing v1.0 (April 2025): https://openssf.org/blog/2025/04/04/launch-of-model-signing-v1-0-openssf-ai-ml-working-group-secures-the-machine-learning-supply-chain/
