# Exercise: Finding and Fixing Weaknesses in AI Code

Course: CAISP (Practical DevSecOps)
Status: Complete (all six steps, including the remediation challenge)

## How to read this document

For the concept with no code, read **Part 1** and **Part 5**. To reproduce the work, read **Parts 2, 3 and 4**.

---

# Part 1: Introduction (for everyone)

## What we are doing

We read an AI application's source code with an automated tool, find places where it does something unsafe, fix them, and confirm the application still works.

This is the companion to the Grype exercise. That one asked "are the **libraries** we use dangerous?" This one asks **"is the code we wrote dangerous?"**

## The idea in plain terms

**Static Application Security Testing (SAST)** analyses source code **without running it**. It reads the code the way a reviewer would, looking for patterns known to be unsafe: a function used insecurely, a password written into the file, a dangerous operation performed on untrusted input.

"Static" is the important word. It never executes anything, so it can inspect code paths that rarely run, and it can be applied before the application is deployed or even finished.

## Why it matters here specifically

The AI code in this lab is only 140 lines and does something completely ordinary: it trains an image classifier and loads the saved model back. Yet the scanner finds three genuine security weaknesses in it, all the same one, and it is **the exact issue that runs through this entire course**: loading a PyTorch model file executes whatever code the file contains.

The Picklescan lab built a malicious file to demonstrate that danger. This lab finds the code that would fall for it, and fixes it in one parameter.

## What to take away

Vulnerable dependencies get attention because they arrive with CVE numbers and severity scores. **Weaknesses in your own code have neither, and are just as exploitable.** A short, unremarkable AI script written by a competent developer contained three, and none of them look wrong to the naked eye.

---

# Part 2: First Principles

**Principle 1: SAST reads patterns, not intentions.**
The tool matches code against known dangerous shapes. It cannot tell whether the file you load is trustworthy; it only knows that `torch.load` on an arbitrary file is capable of executing code. This is why SAST produces false positives, and why every finding needs human judgement about context.

**Principle 2: A weakness is not yet a vulnerability.**
Following the Chapter 5 vocabulary precisely: a **weakness** is a mistake that could contribute to a vulnerability (this is what CWE catalogues, and what SAST finds); a **vulnerability** is an instance an attacker can actually use. `torch.load` without protection is a weakness. It becomes a vulnerability when someone can influence which file gets loaded.

**Principle 3: Severity and confidence are different axes.**
Bandit reports both. Severity is how bad the issue is if real; confidence is how sure the tool is that it is real. A medium severity, high confidence finding is worth more attention than a high severity, low confidence one, because the second may simply be wrong.

**Principle 4: Fixing is not finished until re-scanned and re-tested.**
Identical to the Grype lab, and worth repeating because it is the discipline that distinguishes a security fix from a security gesture. Re-scan proves the weakness is gone; re-test proves you did not break the application removing it.

**Principle 5: A clean scan means "no known patterns matched", not "safe".**
SAST covers the rules it has. Logic flaws, design errors and anything outside the rule set pass silently.

---

# Part 3: Step by Step Replication

## 3.0 Environment

1. Linux (Ubuntu 22.04 in the lab), Python 3.10
2. **Bandit**, a SAST tool specialised for Python source code
3. Target: `caisp-image-classifier`, the same project as the Grype lab and the Chapter 2 fine tuning lab

## 3.1 What SAST needs and produces

**Input:** the source code. Some tools also handle compiled code, bytecode or binaries.

**Output:** a report listing the weaknesses found and recommendations for fixing them.

**The contrast with SCA is worth holding explicitly:**

| | SCA (Grype) | SAST (Bandit) |
| - | ----------- | ------------- |
| Examines | Third party dependencies | Your own source code |
| Finds | Known vulnerabilities (CVE / GHSA) | Weakness patterns (CWE) |
| Identifier | A published advisory | A rule the tool implements |
| Fix | Upgrade a version | Change the code |
| Needs | A dependency manifest | The source |

You need both, because they look at different halves of the same application, and neither sees the other's half.

## 3.2 Install Bandit

```bash
pip install bandit==1.8.5
bandit --help
```

**A small oddity worth noticing:** the command requests 1.8.5 and the install output shows `bandit-1.8.3` being downloaded and installed, with the documentation links in later output also pointing at 1.8.3. Worth verifying the installed version with `bandit --version` before relying on version specific rule behaviour, since which rules exist depends on the version. The lab itself notes that results vary by version.

The help output lists every rule the tool implements, and skimming it is genuinely useful because it tells you what the tool can and cannot see. A sample:

```
B101    assert_used
B102    exec_used
B103    set_bad_file_permissions
B105    hardcoded_password_string
B108    hardcoded_tmp_directory
B113    request_without_timeout
B301    pickle
B614    pytorch_load          ← the one that matters here
```

**Note B301 (`pickle`) and B614 (`pytorch_load`) sitting in the same rule set.** They are the same underlying danger, unsafe deserialisation, and PyTorch earned its own dedicated rule because model loading is now common enough to warrant one.

## 3.3 The target

```bash
git clone https://gitlab.practical-devsecops.training/marudhamaran/caisp-image-classifier.git
cd caisp-image-classifier
cat image-classifier.py
```

The 140 line fine tuning script from Chapter 2: trains a ResNet-18 on five animal classes, saves the model, and loads it back for inference.

## 3.4 The scan

```bash
bandit -r .
```

Three findings, all identical:

```
>> Issue: [B614:pytorch_load] Use of unsafe PyTorch load
   Severity: Medium   Confidence: High
   CWE: CWE-502 (https://cwe.mitre.org/data/definitions/502.html)
   More Info: https://bandit.readthedocs.io/en/1.8.3/plugins/b614_pytorch_load.html
   Location: ./image-classifier.py:113:30
112             # load best model weights
113             model.load_state_dict(torch.load(best_model_params_path))

   Location: ./image-classifier.py:160:20
159             if torch.__version__[:3] == '2.3':
160                 model = torch.load(model_file)

   Location: ./image-classifier.py:175:24
175                     model = torch.load(model_file)

Run metrics:
        Total issues (by severity): Medium: 3
        Total issues (by confidence): High: 3
```

**Reading this properly:**

1. **Three weaknesses, all `B614: pytorch_load`.** Every unprotected `torch.load` call in the file.
2. **Medium severity, high confidence.** Principle 3: Bandit is very sure these calls exist and are unprotected. Whether they are exploitable depends on whether an attacker can control the file, which the tool cannot know.
3. **CWE-502: Deserialization of Untrusted Data.** The same weakness class as the Picklescan lab, now identified formally.

A small documentation inconsistency: the tool output cites **CWE-502**, while Bandit's own B614 documentation page shows **CWE-94** (Code Injection) in its example. Both are defensible descriptions of the same problem (unsafe deserialisation leading to code execution), but CWE-502 is the more precise fit.

## 3.5 The fix

Bandit's documentation names two safe alternatives:

1. **`torch.load` with `weights_only=True`**, which extracts only tensor data and deserialises no arbitrary Python objects. PyTorch then enforces a strict type check, permitting only `torch.Tensor` objects.
2. **The `safetensors` library**, which provides a safe deserialisation mechanism by design.

The lab takes the first, as a minimal in place change:

```bash
sed -i "s/torch.load(\(best_model_params_path\))/torch.load(\1, weights_only=True)/g" image-classifier.py
sed -i "s/torch.load(\(model_file\))/torch.load(\1, weights_only=True)/g" image-classifier.py
```

Which converts, for example:

```python
model.load_state_dict(torch.load(best_model_params_path))
# becomes
model.load_state_dict(torch.load(best_model_params_path, weights_only=True))
```

Verify before re-scanning:

```bash
cat image-classifier.py
```

## 3.6 Re-scan

```bash
bandit -r .
```

```
Test results:
        No issues identified.

Code scanned:
        Total lines of code: 140
Run metrics:
        Total issues (by severity): Undefined: 0, Low: 0, Medium: 0, High: 0
```

Clean. If a loaded model file contains executable code, that code will no longer run, because `weights_only=True` refuses to deserialise anything but tensors.

## 3.7 Re-test

```bash
python3 image-classifier.py
```

Trains 25 epochs, reaches validation accuracy 1.0, saves `sample_image_classifier.pt`. Then:

```bash
python3 image-classifier.py sample-images-for-classification/snake.jpg
```

```
Trying to load model: sample_image_classifier.pt
Hmm. What could this image be?
snake
```

Both halves verified.

## 3.8 Why it still works, which is the interesting part

The lab flags that this is "where things get a little uneasy" and moves on. It is worth understanding, because a naive reading suggests the fix should have broken the program.

Look at what each call loads:

| Line | What is loaded | Compatible with `weights_only=True`? |
| ---- | -------------- | ------------------------------------ |
| 113 | `model.state_dict()`, saved earlier in the script | **Yes.** A state dict is a dictionary of tensors, exactly what the restricted loader permits |
| 160, 175 | The **whole model object**, saved with `torch.save(model, model_file)` | Not on its own |

Lines 160 and 175 load a complete pickled `ResNet` object, not just tensors, so a strict tensors-only load would normally refuse it. It works because of the code already surrounding line 175, from the Chapter 2 lab:

```python
with torch.serialization.safe_globals([
        models.resnet.ResNet, nn.modules.conv.Conv2d, nn.modules.linear.Linear,
        models.resnet.BasicBlock, nn.modules.container.Sequential, ...]):
    model = torch.load(model_file, weights_only=True)
```

**`safe_globals` is an allowlist that extends the restricted loader.** It tells PyTorch: permit tensors, plus these specific named classes I have vetted. So the whole model object can be reconstructed, but only from components on the list, and an attacker's arbitrary callable is not on the list.

That is the good design pattern hiding in this lab: **not "trust everything" or "trust nothing", but "trust an explicit, reviewed list".**

**One caveat worth recording.** Line 160 sits in the `torch.__version__[:3] == '2.3'` branch, which has **no** `safe_globals` wrapper. On that path, `weights_only=True` applied to a whole model object would likely fail. That branch is not taken in this lab environment, so the change was never exercised there. The scan is clean and one code path is untested, which is a small, real illustration of Principle 4: a clean scan plus a passing test only covers the paths the test actually ran. Worth confirming on PyTorch 2.3 before shipping this fix.

---

# Part 4: Security Analysis

## What this exercise establishes

1. **Your own code carries weaknesses too.** The Grype lab found problems in other people's libraries; this one found three in 140 lines of ordinary application code, written without any obvious mistake.
2. **The unsafe pattern is the course's central technical risk.** CWE-502, deserialisation of untrusted data, is the mechanism behind the Picklescan lab, the `trust_remote_code` warnings, "prefer safetensors", "signed is not safe", and the malicious models found on public hubs. This lab is where you fix it in your own code.
3. **The fix is one parameter.** `weights_only=True` closes an arbitrary code execution path with a single keyword argument. Very few security fixes are this cheap, and it is default behaviour in PyTorch 2.6 and later, which is the ecosystem drawing the same conclusion.
4. **Allowlisting is the middle path.** `safe_globals` shows how to keep functionality (loading a whole model) without accepting arbitrary code, by naming exactly what is permitted.
5. **SAST plus SCA is the complete picture.** Neither alone covers an application: one reads your code, the other reads your dependencies.

## Where the weakness becomes a vulnerability

Principle 2 in practice. `torch.load` without protection is dangerous when an attacker can influence the file being loaded, which happens when:

1. The model is downloaded from a public hub (the standard case in this course)
2. The model path is user supplied
3. The model store is writable by more people than you think
4. A model is pulled by floating tag rather than pinned revision, so it can change under you

If the file is generated locally and never leaves the machine, the risk is lower, though defence in depth still argues for the parameter.

## Honest limitations of SAST

1. **Rule coverage defines what is found.** Bandit has around a hundred rules. Anything outside them is invisible, including every logic and design flaw.
2. **No AI specific rules beyond serialisation.** Bandit will not flag an unfiltered prompt path, missing output validation, an over-permissioned tool or an unscoped retrieval query. All the AI specific weaknesses from Chapter 3 pass a Bandit scan cleanly.
3. **False positives are inherent.** The tool cannot assess context, which is why `# nosec` exists and why unreviewed suppressions become a problem of their own.
4. **A clean scan proves nothing about safety.** It proves no implemented rule matched.

## The pipeline version

```bash
bandit -r . -f json -o bandit-report.json      # machine readable for CI
bandit -r . -ll                                 # medium severity and above only
bandit -r . -iii                                # high confidence only
bandit -r . -b baseline.json                    # only new findings vs a baseline
```

The **baseline** flag is the practical one for adopting SAST on an existing codebase: record current findings, then fail the build only on *new* ones, so the gate is useful immediately without requiring the backlog to be cleared first.

---

# Part 5: Conclusion (for everyone)

We scanned 140 lines of unremarkable AI code and found three security weaknesses, all the same one, then removed them with a single keyword argument and confirmed the classifier still recognised a snake.

Two things make this worth more than the small amount of work involved.

The first is what was found. Nothing in that code looked wrong. It loads a model file, which is the most ordinary thing an AI application does, and it does so the way most tutorials show. The weakness is that loading a PyTorch model file **runs whatever code that file contains**, so an application that loads a model from anywhere it does not fully control has an arbitrary code execution path built into its most routine operation. That single fact connects this lab to the malicious pickle we built earlier, to the `trust_remote_code` warnings on every model download, and to the real reverse shell models sitting on public hubs today.

The second is the shape of the fix. `weights_only=True` refuses to reconstruct anything except tensors, and where the application genuinely needs to load a full object, `safe_globals` names an explicit list of vetted classes to permit. Neither blanket trust nor blanket refusal, but a reviewed allowlist. That pattern generalises far beyond PyTorch, and it is the most transferable idea in the exercise.

The habit to carry away is simply that **you should scan your own code, not just your dependencies.** Library vulnerabilities arrive with CVE numbers, severity scores and someone else's fix. The weaknesses you write yourself arrive with none of those, and nobody is going to notify you about them.

---

## Appendix: files touched

| File | Role |
| ---- | ---- |
| `image-classifier.py` | The application scanned and fixed; three `torch.load` calls hardened |
| `sample_image_classifier.pt` | Model produced by the post-fix test run |

## Ideas to take forward

1. Experiment: point the fixed loader at the malicious pickle from the Picklescan lab and confirm `weights_only=True` refuses it rather than executing the payload. This proves the fix end to end.
2. Experiment: rewrite the save and load to use `safetensors` instead, the second option in Bandit's documentation, and re-scan.
3. Experiment: run Bandit against one of the LLM chatbot scripts from Chapter 2 and see how little it flags, demonstrating the AI specific rule gap.
4. Experiment: add `bandit -r . -ll -f json` to a CI step with a baseline, and confirm it fails only on newly introduced findings.
5. Concept file: `concepts/sast-vs-sca.md` on which tool sees which half of an application, and why both are required.
