# Exercise: Analyzing and Fixing Vulnerabilities in Third-Party Components

Course: CAISP (Practical DevSecOps)
Status: Complete (all six steps, including the remediation challenge)

## How to read this document

For the concept with no code, read **Part 1** and **Part 5**. To reproduce the work, read **Parts 2, 3 and 4**.

---

# Part 1: Introduction (for everyone)

## What we are doing

We take a working AI application, scan the third party libraries it depends on, discover that some of them have known security vulnerabilities, upgrade them to safe versions, and confirm the application still works.

This is the single most common, most boring and most valuable supply chain task in all of software security. It is also the one most often skipped.

## The idea in plain terms

Your AI application is a small amount of your code sitting on a large pile of other people's code: `torch`, `torchvision`, `Pillow` and everything those pull in. That pile is where most of your risk lives, because it is code you did not write and rarely look at.

Security researchers continuously find and publish vulnerabilities in popular libraries. When they do, a fixed version is usually released. **Software Composition Analysis (SCA)** is the process of comparing the exact versions you use against that public list of known vulnerabilities, and telling you which of your dependencies are dangerous and what to upgrade to.

## Why it matters

1. **Most exposure is known and already fixed.** These are not zero days. They are published vulnerabilities with published fixes that you have simply not applied yet. From the chapter notes: 96% of vulnerable downloads had a safe version available.
2. **You cannot fix what you cannot see.** When the next Log4Shell lands, the only question that matters is "are we affected, and where?" SCA is how you answer it in minutes.
3. **Fixing is not free.** Upgrading one library can break another, and the lab deliberately walks you into exactly that wall. Managing dependencies is a genuine engineering task, not a one line command.

## What to take away

Scanning is easy; the tool does it in one command. **Remediation is the hard part**, because dependencies constrain each other, security wants the newest version and compatibility wants a specific one, and you cannot declare victory until you have re-scanned *and* re-tested. This lab is really about that loop.

---

# Part 2: First Principles

**Principle 1: Your dependencies are your attack surface.**
The security of your application includes the security of every library it uses, transitively. You inherit their vulnerabilities whether or not you know the libraries exist.

**Principle 2: A vulnerability is a specific version problem.**
"Pillow is vulnerable" is meaningless. "Pillow 10.0.0 has GHSA-3f63-hfp8-52jq, fixed in 10.2.0" is actionable. SCA works by matching your exact installed versions against a database of version specific advisories, which is why version pinning and SCA reinforce each other.

**Principle 3: Fixing one thing can break another.**
Dependencies declare their own required versions of other dependencies. Upgrading `torch` can violate what `torchvision` demands. There is no guarantee that the safe version and the compatible version are the same version, and resolving that tension is the real work.

**Principle 4: Remediation is not complete until re-scanned and re-tested.**
Two independent checks. Re-scanning proves the vulnerability is gone. Re-testing proves you did not break the application while removing it. Skipping either leaves you either insecure or broken, and you will not know which.

**Principle 5: The scan result is a moving target.**
The vulnerability database updates constantly. New advisories appear, and today's safe version may be flagged tomorrow. A scan is a snapshot, not a permanent verdict, which is why SCA belongs in a continuously running pipeline rather than in a one time audit.

---

# Part 3: Step by Step Replication

## 3.0 Environment

1. Linux (Ubuntu 22.04 in the lab), Python 3.10
2. **Grype** (Anchore), an open source SCA / vulnerability scanner
3. Target: the `caisp-image-classifier` project from the Chapter 2 fine tuning lab

## 3.1 What SCA needs, and what it produces

**Inputs**, one or a combination depending on the tool: the source code, the list of dependencies, and sometimes the dependencies actually installed in the system. The third matters for a specific reason the course draws out: when a manifest specifies a **range** rather than an exact version, the tool cannot know which version you will actually get without resolving or installing it. Compare a pinned manifest with a ranged one:

```
   PINNED (unambiguous)      RANGED (needs resolution to scan)
   ────────────────────      ─────────────────────────────────
   torch==2.3.0              Flask>=2.2.0            # 2.2.0 or newer
   torchvision==0.18.0       requests>=2.25.0        # 2.25.0 or newer
   Pillow==10.0.0            numpy~=1.21.0           # >=1.21.0, <1.22.0
                             "lodash": "^4.17.0"     # >=4.17.0, <5.0.0
                             "axios": "~1.3.0"       # >=1.3.0,  <1.4.0
```

With ranges, the tool must install or resolve to know the real version to check. Pinned exact versions remove that ambiguity, which is one more reason pinning and SCA reinforce each other.

**Output**: a report listing the dependencies, the vulnerabilities found in them, and the recommended fixed versions.

## 3.2 Install Grype

```bash
curl -sSfL https://raw.githubusercontent.com/anchore/grype/main/install.sh | sh -s -- -b /usr/local/bin
```

Grype scans container images, filesystems and SBOMs. It reads what packages are present, then matches them against its vulnerability database.

**A note worth making about the install command itself:** `curl | sh` pipes a remote script straight into a shell, executing whatever that URL currently returns. In a lab this is fine; in a real pipeline it is exactly the kind of unverified remote execution the rest of this chapter warns against. The more defensible pattern is to download, pin to a known release, verify a checksum or signature, then run.

## 3.3 The target and its dependencies

```bash
git clone https://gitlab.practical-devsecops.training/marudhamaran/caisp-image-classifier.git
cd caisp-image-classifier
cat requirements.txt
```

```
torch==2.3.0
torchvision==0.18.0
Pillow==10.0.0
```

Three dependencies, all pinned to exact versions. This is the fine tuning lab's application from Chapter 2, reused as a realistic scan target.

## 3.4 The first scan

```bash
grype .
```

On first run Grype downloads its vulnerability database (around 83 MB), then scans:

```
 ✔ Scanned for vulnerabilities     [7 vulnerability matches]
   ├── by severity: 2 critical, 3 high, 1 medium, 1 low, 0 negligible
   └── by status:   5 fixed, 2 not-fixed, 0 ignored

NAME    INSTALLED  FIXED-IN  TYPE    VULNERABILITY        SEVERITY  EPSS%  RISK
pillow  10.0.0     10.0.1    python  GHSA-j7hp-h8jx-5ppr  High      99.87  85.6 (kev)
torch   2.3.0      2.6.0     python  GHSA-53q9-r3pm-6pq6  Critical  62.30   0.4
pillow  10.0.0     10.2.0    python  GHSA-3f63-hfp8-52jq  Critical  60.24   0.3
pillow  10.0.0     10.3.0    python  GHSA-44wm-f244-xhp3  High      27.82  <0.1
torch   2.3.0                python  GHSA-3749-ghw9-m3mg  Low        4.58  <0.1
torch   2.3.0                python  GHSA-887c-mr87-cxwp  Medium     2.45  <0.1
pillow  10.0.0     10.0.1    python  GHSA-56pw-mpj4-fxww  High        N/A   N/A
```

Reading this properly is the skill:

1. **Seven vulnerabilities across two packages.** `pillow` and `torch` are vulnerable; `torchvision` is clean.
2. **Severity** is the intrinsic badness. Two critical, three high.
3. **FIXED-IN** tells you the version that resolves it. Where it is blank (the two `torch` low and medium), **no fix exists yet**, which is a different situation: you cannot upgrade your way out.
4. **EPSS%** is the Exploit Prediction Scoring System: the probability the vulnerability will be exploited in the wild. The `pillow` GHSA-j7hp-h8jx-5ppr sits at 99.87%, marked `(kev)`, meaning it is on CISA's Known Exploited Vulnerabilities list. **This is the one to fix first**, not because its severity is highest but because it is actively being exploited right now.
5. **RISK** is Grype's combined prioritisation (severity plus EPSS plus KEV). Sorting by risk is why that same `pillow` entry is at the top: 85.6 versus everything else under 0.5.

**The lesson hidden in the EPSS column:** a "high" that is actively exploited (EPSS 99.87%) is more urgent than a "critical" that is not (EPSS 0.3%). Severity alone would have you fix the critical first. Real prioritisation uses exploitability, which is what EPSS and KEV add.

## 3.5 The challenge: remediate

The task: update `requirements.txt` to non vulnerable versions, fixing at least the high and critical findings, then re-scan and re-test.

### First attempt

```bash
cat > requirements.txt <<EOF
torch==2.7.1
torchvision==0.18.0
Pillow==12.1.1
EOF
```

Grype recommended `torch` 2.6.0 and `pillow` 10.3.0, but the write up deliberately reaches for the latest (`torch` 2.7.1, `pillow` 12.1.1) to fix everything comprehensively rather than the single advisory each fix targets.

Re-scanning `requirements.txt` looks clean of criticals and highs. But then:

```bash
pip install -r requirements.txt
```

```
ERROR: Cannot install -r requirements.txt (line 2) and torch==2.7.1
because these package versions have conflicting dependencies.

The conflict is caused by:
    The user requested torch==2.7.1
    torchvision 0.18.0 depends on torch==2.3.0

ERROR: ResolutionImpossible
```

**This is the whole point of the lab.** The scan said the versions were safe. The install said they were incompatible. `torchvision` 0.18.0 pins `torch` 2.3.0, the exact version we are trying to move away from for security. Security and compatibility are pulling in opposite directions. Principle 3, in a real error message.

### Resolving the conflict

The fix is to find a `torchvision` that is compatible with the `torch` you need, rather than forcing an old one. `pip` can discover this:

```bash
pip install torch==2.7.1 torchvision --upgrade
```

Letting `torchvision` float to whatever version matches `torch` 2.7.1, pip resolves it to **torchvision 0.22.1**. Now pin that discovered pair:

```bash
cat > requirements.txt <<EOF
torch==2.7.1
torchvision==0.22.1
Pillow==12.1.1
EOF

pip install -r requirements.txt
```

Installs cleanly. **Note what happened here methodologically:** we loosened the pin *temporarily* to let the resolver find a compatible version, then re-pinned to the exact version it found. We did not leave the range in place. Loosening is a discovery technique; the end state is still pinned, for the reproducibility and security reasons from the pinning lesson.

## 3.6 Re-scan (Principle 4, part one)

```bash
grype .
```

```
 ✔ Scanned for vulnerabilities     [1 vulnerability matches]
   ├── by severity: 0 critical, 0 high, 1 medium, 0 low, 0 negligible

NAME   INSTALLED  TYPE    VULNERABILITY        SEVERITY  RISK
torch  2.7.1      python  GHSA-887c-mr87-cxwp  Medium    <0.1
```

Seven down to one. No criticals, no highs. The remaining medium in `torch` 2.7.1 has **no fix available**, so it cannot currently be remediated by upgrading; it is documented and accepted rather than fixed, which is a legitimate outcome. The lab's target was high and critical, and that is met.

## 3.7 Re-test (Principle 4, part two)

Removing vulnerabilities is worthless if the application no longer runs. The classifier is the fine tuning script from Chapter 2:

```bash
python3 image-classifier.py
```

It trains through 25 epochs, reaches validation accuracy of 1.0, and saves `sample_image_classifier.pt`. Then inference:

```bash
python3 image-classifier.py sample-images-for-classification/snake.jpg
```

```
Trying to load model: sample_image_classifier.pt
Hmm. What could this image be?
snake
```

Works. The dependencies moved from `torch` 2.3.0 to 2.7.1 and `pillow` 10.0.0 to 12.1.1, two and eleven major-ish versions respectively, and the application still trains and classifies correctly. **Only now is the remediation complete**, because both checks passed.

---

# Part 4: Security Analysis

## What this exercise establishes

1. **SCA is the answer to "are we affected?"** and it is fast and cheap. There is no excuse not to run it.
2. **Prioritisation needs exploitability, not just severity.** EPSS and KEV are why the actively exploited high outranks the theoretical critical. Fixing strictly by severity wastes effort on things nobody is exploiting while leaving a KEV entry in place.
3. **Remediation is constrained, not free.** The `torch`/`torchvision` conflict is the real world every time: the safe version and the compatible version are not automatically the same, and reconciling them is engineering work.
4. **Some vulnerabilities have no fix.** The residual `torch` medium cannot be upgraded away. The correct response is to document and accept it (or mitigate around it), not to pretend it is gone. This is risk acceptance from the Chapter 5 treatment options, applied concretely.
5. **Verification is two sided.** Re-scan for security, re-test for function. Neither alone is sufficient.

## Where this sits in the AI context

The vulnerable packages are `torch` and `pillow`, the ordinary plumbing of every AI project. This reinforces the point from the scanning lesson: **an AI system's real vulnerability profile is mostly conventional CVEs in conventional libraries.** Prompt injection is the exotic risk that gets attention, but the thing an SCA scan actually catches is a buffer overflow in an image library and a deserialization issue in a tensor library. Both matter, and only one is novel.

Worth naming: several of these findings are themselves the pickle and deserialization risks from elsewhere in the chapter. `torch` vulnerabilities frequently concern unsafe model loading, which is the same theme as the Picklescan lab from a different angle.

## Honest limitations

1. **SCA finds known vulnerabilities only.** A malicious package with no CVE, a poisoned model or a backdoor passes a Grype scan cleanly. This is dependency *vulnerability* management, not dependency *trust* management, and the two are different problems.
2. **The scanned manifest is not always the installed reality.** Grype scanned `requirements.txt`; what actually runs is what pip resolved and installed. Scanning the installed environment (or a lockfile, or the built container) is more truthful than scanning the manifest. The first-attempt clean scan on an uninstallable manifest is proof of this gap.
3. **Results drift.** The lab flags this repeatedly: today's fix version may be tomorrow's vulnerable version. A scan is a snapshot, which is the argument for continuous scanning in CI rather than a periodic manual pass.

## The mitigation this lab is teaching

The full loop, which belongs in a pipeline, not a person's memory:

```
   pin exact versions ──► scan ──► prioritise by risk (severity + EPSS + KEV)
        ▲                                    │
        │                                    ▼
   re-pin the fix ◄── re-test ◄── re-scan ◄── upgrade to compatible safe version
```

Run it on every build, fail the build on new high or critical findings that have a fix, and track the ones that do not.

---

# Part 5: Conclusion (for everyone)

We scanned an AI application, found seven known vulnerabilities in its libraries, and reduced them to a single unfixable medium, all while keeping the application working.

The scanning was trivial: one tool, one command, a clear report. Everything interesting happened afterwards. The report told us `pillow`'s actively exploited flaw mattered more than a higher rated one nobody is exploiting, which is a lesson severity alone would have hidden. The obvious fix, upgrade everything to the latest, ran straight into a dependency conflict, because the safe version of `torch` was incompatible with the pinned version of `torchvision`, and reconciling that took real work rather than a flag. And the job was not done when the scan came back clean; it was done when the scan came back clean *and* the classifier still recognised a snake.

That is the shape of dependency security in practice, for AI systems and everything else. The vulnerabilities are mostly known and already fixed upstream, so the value is entirely in the discipline of finding them, prioritising by what is actually being exploited, upgrading without breaking anything, and proving both halves of that with a re-scan and a re-test. It is unglamorous, it is continuous, and it catches more real risk than most of the exotic techniques elsewhere in this course.

---

## Appendix: files touched

| File | Role |
| ---- | ---- |
| `requirements.txt` | Pinned dependency manifest, edited from vulnerable to safe versions |
| `image-classifier.py` | The application under test (from the Chapter 2 fine tuning lab) |
| `sample_image_classifier.pt` | Model produced by the post-remediation test run |

## Ideas to take forward

1. Experiment: scan the *installed environment* or a generated SBOM instead of `requirements.txt`, and compare, to see the manifest versus reality gap concretely.
2. Experiment: generate an SBOM with Syft, then feed it to Grype (`syft . -o json | grype`), connecting this lab to the SBOM lessons.
3. Experiment: add `grype . --fail-on high` to a CI step and watch it break the build on a reintroduced vulnerable version.
4. Experiment: document the residual unfixable `torch` medium as a formal risk acceptance, with owner and review date, per the Chapter 5 treatment model.
5. Concept file: `concepts/sca-and-remediation.md` on the scan, prioritise (EPSS/KEV), upgrade, reconcile, re-scan, re-test loop.
