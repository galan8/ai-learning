# Exercise: Scanning a Malicious Pickle File using Picklescan

Course: CAISP (Practical DevSecOps)
Status: Complete

## How to read this document

For the concept with no code, read **Part 1** and **Part 5**. To reproduce the work, read **Parts 2, 3 and 4**.

---

# Part 1: Introduction (for everyone)

## What serialization and deserialization are

Software works with objects in memory: a trained model, a configuration, a data structure. Those objects exist only while the program runs. To save one to a file, send it over a network, or hand it to another program, it has to be turned into a stream of bytes. That conversion is **serialization**. Turning the bytes back into a live object is **deserialization**.

```
   live object  ──serialize──►  bytes on disk / wire  ──deserialize──►  live object
   (in memory)                  (a file, a download)                    (in memory)
```

Every model file you have loaded in this course was deserialized. `torch.load`, `from_pretrained`, loading a `.pkl`: all of them take bytes produced elsewhere and reconstruct an object in your process.

## Why deserialization is dangerous

Here is the whole security problem in one sentence: **deserializing data means letting a file, which someone else may have created, decide what your program builds in memory.**

For a simple format that only stores data (a number, a string, a list), that is safe. The worst a malicious file can do is give you wrong data. But some serialization formats do more than store data. They store **instructions for how to rebuild the object**, and rebuilding can involve running code. When that is the case, opening the file runs the instructions, and the instructions can be anything.

This is why the deserialization of untrusted data is a recognised vulnerability class in its own right (CWE-502), and why it appears throughout this course: it is the mechanism behind the MITRE ATLAS "User Execution" technique, the reason `trust_remote_code=True` carries a warning, and the reason safetensors was created.

## Python pickle specifically

**A pickle file is a binary file created by Python's `pickle` module**, which serializes and deserializes Python objects. It usually has a `.pkl` extension, though PyTorch's `.pt`, `.pth` and `.bin` model files are pickle underneath. The contents are binary and cannot be read without unpickling.

Pickle is the dangerous kind of format. To reconstruct an object it may call a method the object defines called `__reduce__`, which returns **a callable and its arguments**. Pickle then calls that callable. Nothing constrains what the callable is. If a malicious file says "to rebuild me, call `eval` with this string", then unpickling it runs `eval` on that string, and the code executes on your machine **before you have done anything with the object at all**.

### The disadvantages of pickle, stated plainly

1. **It can execute arbitrary code** if maliciously crafted. Never load a pickle file from an untrusted source.
2. **It does not verify integrity or authenticity.** Pickle loads whatever it is given, with no check that the data is what it claims to be or that it has not been tampered with.
3. **It is Python only.** The format is not interoperable with other languages, unlike JSON.
4. **It can be large** compared to formats like JSON or MessagePack.
5. **It can overwrite objects in memory during deserialization**, producing unexpected or malicious behaviour.

## What this exercise does

We build a deliberately malicious pickle file, confirm it would execute code on load, and then scan it with **Picklescan**, a tool that inspects a pickle file *without* deserialising it and flags the dangerous instructions inside. We then point the same scanner at a real Hugging Face model that carries the same trick.

## Why it matters

Every model you download in a pickle based format is a program you are about to run. You cannot tell by looking, because the file is binary and the dangerous part is a small instruction buried in it. Scanning before loading, and preferring formats that cannot execute code, is the difference between downloading data and running a stranger's code with your permissions.

---

# Part 2: First Principles

**Principle 1: Loading a pickle is running a program.**
Deserialising a pickle can invoke `__reduce__`, which names a callable to run. Opening the file is therefore equivalent to executing whatever that callable does. The verb "load" hides an "execute".

**Principle 2: The danger is in the format, not the file's stated purpose.**
A file called `pytorch_model.bin` is trusted because of its name and extension. The name is not a security property. A model file and a payload can be the same file.

**Principle 3: You can inspect the instructions without executing them.**
Pickle is a stack based instruction stream. A scanner can read those instructions and see which callables the file will invoke, without ever running them. This is what makes safe scanning possible: you read the recipe without cooking it.

**Principle 4: Scanning detects known dangerous imports, not all malice.**
Picklescan flags calls to known dangerous callables such as `eval`, `exec`, `os.system` and `__builtin__` functions. An attacker who reaches the same effect through a callable not on the list, or who compresses the file to evade the reader, is not caught. Scanning raises the cost; it does not close the door.

---

# Part 3: Step by Step Replication

## 3.0 Environment

1. Linux, Python 3.10
2. `picklescan==0.0.20`

## 3.1 Setup

```bash
apt update
apt install -y python3 python3.10-venv python3-pip

pip install picklescan==0.0.20
```

## 3.2 Create a malicious pickle

```python
# create_malicious_pickle.py
import pickle

class MaliciousCode:
    def __reduce__(self):
        # Whatever this returns is what pickle will CALL on load.
        # Here: call eval() on a string. eval runs arbitrary Python.
        return (eval, ("print('Malicious Code Executed')",))

malicious_data = MaliciousCode()

with open('malicious.pkl', 'wb') as f:
    pickle.dump(malicious_data, f)

print("Malicious pickle file 'malicious.pkl' created.")
```

```bash
python3 create_malicious_pickle.py
# Malicious pickle file 'malicious.pkl' created.
```

**The mechanism, in the one method that matters.** `__reduce__` returns a tuple of `(callable, args)`. When pickle deserialises this object, it calls `callable(*args)`. This file returns `(eval, ("print('Malicious Code Executed')",))`, so loading it runs `eval("print('Malicious Code Executed')")`.

The payload here is a harmless `print`. **That is the only thing that is harmless about it.** The exact same structure with `os.system("...")`, a reverse shell, or code to read your SSH keys would execute identically. The demonstration is deliberately benign so that the *structure* is the lesson, not the payload. This is the same pattern JFrog found in around a hundred real models on Hugging Face in March 2024, one of which opened a reverse shell on load.

## 3.3 Scan the file

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

Picklescan read the pickle's instruction stream, saw that loading it would import and call `builtins.eval`, and flagged it. **Crucially, it did this without deserialising the file**, so the malicious code never ran during the scan. That is the entire value of the tool: it lets you find out whether a file is dangerous without triggering the danger.

## 3.4 Scan a real Hugging Face model

The same trick exists in the wild. Yannic Kilcher published a deliberately named demonstration model for exactly this teaching purpose:

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

Two things to take from this:

1. **The payload is inside `pytorch_model.bin`.** A file with the most ordinary model name imaginable, containing a pickle (`archive/data.pkl`) with an `eval` call. Nothing about the filename or the repository signals danger; the model card and interface look normal. The name "totally-harmless-model" is the author's joke, not a label attackers would use.
2. **Picklescan reaches into the model archive** (PyTorch model files are zip archives containing a pickle) and scans the pickle inside. This is the same capability Hugging Face now runs automatically on uploads.

## 3.5 Results summary

| Target | Result |
| ------ | ------ |
| Locally crafted `malicious.pkl` | Dangerous import `builtins eval` found, without executing it |
| `ykilcher/totally-harmless-model` on Hugging Face | Dangerous import `__builtin__ eval` found inside `pytorch_model.bin` |

Both detections happened without deserialisation, which is the point.

---

# Part 4: Security Analysis

## Where this sits in the course

This exercise is the concrete demonstration of a thread running through every chapter:

1. **Chapter 2 fine tuning lab:** `torch.save` and `torch.load` are pickle, and the safe path used `weights_only` and `safe_globals`.
2. **Chapter 3 supply chain:** model file formats are executable code; safetensors exists to stop this.
3. **Chapter 6 model signing and scanning:** artifact scanning with `modelscan` and Hugging Face's built in pickle scanning is this tool's capability at platform scale.

It is the same vulnerability (CWE-502, deserialisation of untrusted data) that gives non AI systems remote code execution, applied to the artifact at the centre of every AI system.

## What scanning does and does not buy

| Property | Reality |
| -------- | ------- |
| Detects known dangerous callables | Yes: `eval`, `exec`, `os.system`, `__builtin__` functions and similar |
| Executes the payload during scanning | No, and that is the whole value |
| Detects a backdoor in the weights | No. A backdoor lives in the numbers, not in a dangerous import |
| Defeated by evasion | Yes. The nullifAI technique (ReversingLabs, Feb 2025) evaded pickle scanning by compressing malicious PyTorch models with 7z instead of zip, so the scanner could not read the pickle |

**Scanning is detection, not prevention.** It raises the attacker's cost and catches the unsophisticated cases, which is worth doing, but it is one layer.

## The controls, in order of strength

1. **Prefer safetensors.** The durable fix. A format that stores only tensors and cannot execute code removes the vulnerability class rather than detecting instances of it. `eval` in a safetensors file is impossible, not merely detectable.
2. **Load with `weights_only=True`** for PyTorch checkpoints you cannot avoid loading, which restricts what deserialisation may reconstruct.
3. **Scan before loading**, in the pipeline, as a gate. Picklescan and modelscan both fit here. Verification that is optional does not happen.
4. **Verify provenance and signature** so you know who produced the file, which is the model signing lesson.
5. **Sandbox any load of an untrusted pickle** with no credentials, no network and no filesystem access, so that if code does execute it reaches nothing.

**The order matters.** Safetensors makes the other four unnecessary for that file. Scanning is what you do when you are stuck with pickle.

## The trap to avoid

The most dangerous habit this exercise guards against is **loading a model to inspect it.** The instinct on receiving a suspicious model is to open it and look. With a pickle, opening it *is* the exploit. Scan first, always, and never deserialise something you are unsure about outside a sandbox.

---

# Part 5: Conclusion (for everyone)

We made a file that runs code the moment it is opened, confirmed a tool could spot it without opening it, and then found the identical trick sitting in a normal looking model on a public hub.

The lesson is about a word. When you "load" a model, you are not reading data, you are running a program, if that model is in a pickle based format. The format decides. A file named `pytorch_model.bin` earns trust from its name, and the name is worth nothing, because the model and the payload can be the same bytes.

Two habits follow, and they are small. First, **prefer formats that cannot execute code**: safetensors exists precisely so that loading a model is reading data and nothing more. Second, when you are stuck with a pickle, **scan it before you load it**, because a scanner can read the file's intentions without carrying them out, and that gap between reading and executing is the only safe place to make a decision.

The uncomfortable part is that scanning is not a guarantee. It catches the obvious `eval`, and it was evaded within a year by someone who simply zipped the file differently. So scanning is a layer, not an answer, and the answer is to not use a format that can betray you. Where you cannot avoid it, scan, verify who made it, and if you must open something you do not trust, open it somewhere it can do no harm.

---

## Appendix: files created

| File | Role |
| ---- | ---- |
| `create_malicious_pickle.py` | Builds a pickle whose `__reduce__` calls `eval` on load |
| `malicious.pkl` | The resulting malicious pickle, for scanning |

## Ideas to take forward

1. Experiment: convert a small model to safetensors and confirm that no `__reduce__` payload can be embedded, contrasting it with the pickle version.
2. Experiment: run `modelscan` against the same files and compare its output and coverage with Picklescan.
3. Experiment (carefully, in a sandbox): replace the benign `print` payload with something observable but harmless, such as writing a file to `/tmp`, and confirm it executes on load. Demonstrates Principle 1 without real harm.
4. Concept file: `concepts/insecure-deserialisation.md` on CWE-502, why pickle is unsafe, and why safetensors solves it rather than mitigating it.
