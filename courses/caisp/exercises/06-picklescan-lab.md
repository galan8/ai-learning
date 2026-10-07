# Exercise: Scanning a Malicious Pickle File using Picklescan

Course: CAISP (Practical DevSecOps)
Status: Complete

## How to read this document

For the concept with no code, read **Part 1** and **Part 5**. To reproduce the work, read **Parts 2, 3 and 4**.

---

# Part 1: Introduction (for everyone)

## What we are doing

We are building a deliberately malicious model file, confirming it runs code the moment it is loaded, and then using a scanner to catch it before that happens. The point is to see, concretely, why "download the model and load it" is a dangerous default.

## The idea in plain terms

Most people picture a machine learning model as a big block of numbers, weights, and nothing more. That is true for some file formats and dangerously false for the most common one.

Python has a built in way to save any object to a file, called **pickle**, and it is what PyTorch's traditional `.pt`, `.pth` and `.bin` files are built on. The catch is that a pickle file does not just store data; it stores **instructions for rebuilding an object**, and those instructions can include arbitrary code. So loading a pickle file can *run* code, on your machine, before you have done anything with the model at all.

That means a model file is not data you open. It is a program you execute. Downloading one from a stranger and loading it is the same category of act as downloading a stranger's executable and running it.

## Why it matters

1. **The dangerous moment is loading, not using.** The code runs during deserialisation, before the first prediction. By the time you notice anything is wrong, it has already run.
2. **The file can look completely normal.** A poisoned model carries an ordinary model card, ordinary metrics, an ordinary name. Nothing on the page warns you.
3. **A signature does not save you here.** A validly signed pickle file is still a pickle file. Signing proves who published it and that it was not altered; it does not make executing it safe.

## What to take away

If you did not produce a model file yourself, treat it as untrusted code, because that is what a pickle based one is. Scan it before loading, prefer formats that cannot execute code (safetensors), and never let a pipeline load an unscanned model from an external source.

---

# Part 2: First Principles

**Principle 1: Serialisation that can rebuild any object can run any code.**
Pickle's job is to reconstruct arbitrary Python objects. To do that it has to be able to call functions during loading. The `__reduce__` method tells pickle how to rebuild an object, and it returns a callable plus arguments. Nothing constrains that callable to be safe, so it can be `eval`, `os.system`, or anything else.

**Principle 2: The payload executes on load, not on use.**
`pickle.load()` runs the reconstruction instructions immediately. There is no "inspect first" step. This is why the risk cannot be managed by being careful with the model afterwards; the damage is done at load time.

**Principle 3: The scanner reads the instructions without following them.**
Picklescan works because a pickle file's opcodes can be parsed statically. It can see that the file *intends* to import and call `eval` without ever executing that intent. It inspects the recipe rather than cooking it.

**Principle 4: Static detection is a blocklist, and blocklists are evadable.**
The scanner flags known dangerous imports (`eval`, `exec`, `os.system`, `__builtin__` calls and similar). An attacker who reaches a dangerous capability through a name the scanner does not recognise slips past. This is why scanning reduces risk rather than removing it, and why a safe format beats a scanned unsafe one.

---

# Part 3: Step by Step Replication

## 3.0 Environment

1. Linux (Ubuntu 22.04), Python 3.10
2. `picklescan==0.0.20`

## 3.1 Setup

```bash
apt update
apt install -y python3 python3.10-venv python3-pip

pip install picklescan==0.0.20
```

## 3.2 Build a malicious pickle file

The whole attack is the `__reduce__` method. When pickle serialises this object it records "to rebuild me, call `eval` with this string", and when anything later loads the file, `eval` runs.

```python
# create_malicious_pickle.py
import pickle

class MaliciousCode:
    def __reduce__(self):
        # Whatever this returns is executed on unpickling.
        # Here it prints a message; it could run any code.
        return (eval, ("print('Malicious Code Executed')",))

malicious_data = MaliciousCode()

with open('malicious.pkl', 'wb') as f:
    pickle.dump(malicious_data, f)

print("Malicious pickle file 'malicious.pkl' created.")
```

```bash
python3 create_malicious_pickle.py
```

**What to understand here:** the payload is a harmless `print`. That is deliberate and it is the safe way to demonstrate the mechanism. The `eval` call would run *anything* put in that string: opening a reverse shell, reading credentials, encrypting files. The demonstration is benign; the capability is total.

**Optional, to feel the risk viscerally:** loading the file executes the payload.

```bash
python3 -c "import pickle; pickle.load(open('malicious.pkl','rb'))"
# prints: Malicious Code Executed
```

Nothing was done with a model. The mere act of `pickle.load` ran the code. That single line is the entire lesson.

## 3.3 Scan the local file

```bash
picklescan --path malicious.pkl
```

```
/root/malicious.pkl: dangerous import 'builtins eval' FOUND
----------- SCAN SUMMARY -----------
Scanned files: 1
Infected files: 1
Dangerous globals: 1
```

The scanner detected the intent to call `eval` **without running it**. Principle 3 in action: it parsed the opcodes, saw a dangerous import, and reported it, all statically.

## 3.4 Scan a model on Hugging Face

Picklescan can scan a remote model directly, which is the realistic use: check before you download and load.

```bash
picklescan --huggingface ykilcher/totally-harmless-model
```

```
https://huggingface.co/ykilcher/totally-harmless-model/resolve/main/pytorch_model.bin:archive/data.pkl: dangerous import '__builtin__ eval' FOUND
----------- SCAN SUMMARY -----------
Scanned files: 1
Infected files: 1
Dangerous globals: 1
```

Two things worth noticing:

1. **The model is named "totally-harmless-model".** It is a well known teaching artifact (by Yannic Kilcher) that is deliberately not harmless. The name is the joke and the lesson: **the file name and the model card tell you nothing about safety.**
2. **The malicious pickle is inside `pytorch_model.bin`.** A `.bin` file is a zip archive containing a `data.pkl`. This is exactly the format every early lab in this course loaded with `from_pretrained`. The danger is not exotic; it is the default file format of the ecosystem.

## 3.5 What a pickle file is, and why it is used at all

A **pickle file** (`.pkl`) is a binary serialisation of Python objects: it converts objects to a byte stream (pickling) and reconstructs them later (unpickling). It is convenient because it can serialise almost any Python object with one function call, which is why PyTorch adopted it for saving models.

Its disadvantages are the reason this lesson exists:

1. **Arbitrary code execution.** A crafted pickle runs code on load. Never load one from an untrusted source.
2. **No integrity or authenticity checking.** Pickle does not verify what it is loading came from where it claims or is unmodified.
3. **Python only.** The format is not interoperable with other languages.
4. **Opaque and large.** Binary, not human readable, and bigger than formats like JSON.

## 3.6 Results summary

| Step | Outcome |
| ---- | ------- |
| Build malicious pickle | `malicious.pkl` created with an `eval` payload in `__reduce__` |
| Load it | Payload executes immediately on `pickle.load` |
| Scan local file | Picklescan reports `dangerous import 'builtins eval' FOUND` without executing it |
| Scan Hugging Face model | Same detection inside a `.bin` model file named "totally-harmless-model" |

---

# Part 4: Security Analysis

## What this maps to

This is **insecure deserialisation** (CWE-502) applied to model files, and it is the concrete mechanism behind two things from earlier in the course:

1. The MITRE ATLAS **User Execution** technique (Chapter 2 and 5): executable code serialised into a model, running on deserialisation.
2. The Chapter 6 supply chain point that **a model file is executable code, and signed is not safe.**

It is not theoretical. JFrog found around a hundred malicious models on Hugging Face in early 2024, one of which opened a reverse shell on load. In February 2025 ReversingLabs disclosed the **nullifAI** technique, which evaded Hugging Face's pickle scanning by compressing the malicious model with 7z instead of the expected zip. That last case is Principle 4 made real: the scanner is a blocklist, and blocklists get bypassed.

## Where picklescan fits, and where it does not

**What it does well:** catches known dangerous imports statically, before load, locally or against a remote model. Cheap, fast, scriptable, and a sensible CI gate.

**What it cannot do:**

1. **Detect novel evasions.** nullifAI is the proof. A dangerous capability reached through an unrecognised path passes.
2. **Detect a backdoor.** A backdoored model has entirely clean pickle contents. The malicious behaviour is in the weights, not in the serialisation. Picklescan sees a healthy file. This is the same blind spot as behavioural model scanning, and it is why provenance, not scanning, is the control against backdoors.
3. **Guarantee safety.** A clean picklescan result means "no known dangerous imports found", not "safe to load".

## The control hierarchy

In order of strength:

1. **Prefer safetensors.** A format that stores only tensors and cannot execute code removes the entire class of attack rather than trying to detect it. This is the real fix; everything below is mitigation for when you cannot use it.
2. **Scan before loading**, as a pipeline gate, not a manual habit. Picklescan or `protectai/modelscan`.
3. **Load with `weights_only=True`** on newer PyTorch, which restricts what deserialisation may reconstruct.
4. **Pin the model revision** by commit hash, so you scan and load the exact version you reviewed.
5. **Verify provenance and signatures**, remembering that this addresses authenticity, not deserialisation safety.
6. **Sandbox any load of an untrusted model** with no credentials, no network and no filesystem access, so that if something does execute, it reaches nothing.

**The ordering is the lesson.** Scanning sits second because detection is always a step behind evasion. The first control eliminates the mechanism; the rest reduce the odds.

## Why "signed is not safe" is the sharpest point

A publisher could sign this exact malicious model with a valid key. The signature would verify perfectly: authentic publisher, unmodified file. And it would still run `eval` on load. Authenticity, integrity and safety are three separate properties. Signing gives you the first two. Only a safe format or a sandbox gives you the third.

---

# Part 5: Conclusion (for everyone)

We wrote a model file that runs code the instant it is opened, watched it do exactly that, and then caught it with a scanner that reads the file's intentions without carrying them out.

The uncomfortable takeaway is how ordinary the dangerous file is. It was not a strange format or an exotic exploit. It was `pytorch_model.bin`, the default way models were saved and loaded throughout the first half of this course, and the malicious version was indistinguishable from a normal one by name, by model card, or by eye. The only way to know was to scan it, and even the scan only tells you it found nothing it recognised as dangerous, which is not the same as safe.

So the practical conclusion is a habit and a preference. The habit: never load a model file you did not produce without scanning it first, ideally as an automated gate rather than a thing you remember to do. The preference: use safetensors wherever you can, because a format that physically cannot execute code beats any amount of scanning a format that can. And the caveat that ties the whole chapter together: a signature tells you who made the file and that nobody changed it, and tells you nothing at all about whether loading it will hurt you.

---

## Appendix: files created

| File | Role |
| ---- | ---- |
| `create_malicious_pickle.py` | Builds a pickle whose `__reduce__` runs `eval` on load |
| `malicious.pkl` | The malicious artifact, for scanning |

## Ideas to take forward

1. Experiment: convert a small real model to safetensors, attempt to embed a payload, and confirm the format gives it nowhere to go.
2. Experiment: run `protectai/modelscan` against the same files and compare its findings and output to picklescan's.
3. Experiment: wrap picklescan into a pre load gate in a script that refuses to call `from_pretrained` until a scan passes, turning the manual step into an enforced control.
4. Concept file: `concepts/pickle-and-model-serialisation.md` on why `.pt`/`.bin` execute code, what safetensors changes, and why scanning is second to format choice.
