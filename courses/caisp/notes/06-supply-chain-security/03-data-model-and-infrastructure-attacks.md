# Data, Model, and Infrastructure Based Attacks

The course groups AI supply chain attacks into four columns. This is the taxonomy worth memorising, because it maps cleanly onto where you place controls.

```
┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐ ┌──────────────────┐
│ DATA             │ │ MODEL            │ │ INFRASTRUCTURE   │ │ DEVELOPMENT      │
├──────────────────┤ ├──────────────────┤ ├──────────────────┤ ├──────────────────┤
│ Data theft       │ │ Model theft      │ │ Hardware         │ │ IDEs             │
│ Data poisoning   │ │ Model editing    │ │ Network          │ │ CI/CD systems    │
│ Foundation model │ │ Backdoors        │ │ Software         │ │ Test suites      │
│ Fine tuned model │ │                  │ │ APIs             │ │                  │
│ RAG              │ │                  │ │                  │ │                  │
│ User inputs      │ │                  │ │                  │ │                  │
└──────────────────┘ └──────────────────┘ └──────────────────┘ └──────────────────┘
```

## 1. Data based attacks

### Where poisoning can enter

The data column lists four distinct injection points, ordered by how reachable each is:

1. **Foundation model training.** Enormous corpora, expensive to influence, but the payoff reaches every downstream user. Requires resources few attackers have.
2. **Fine tuned model training.** Much smaller dataset, so proportionally less poisoned material is needed for the same effect. This is the realistic target.
3. **RAG.** Documents retrieved at query time. Strictly not training data, since weights never change, but the same outcome by a different route, and usually the easiest of the four to modify.
4. **User inputs.** Any system that learns from what users say can be steered by what users say. Tay remains the standing example: coordinated abusive input produced offensive output within roughly a day. Some frontier providers do train on user prompts, which makes this live rather than historical.

**The practical implication:** if you are testing, target the layer you can reach. Poisoning a foundation model is not a realistic test; poisoning a fine tuning set or a RAG corpus is.

### Data theft

Training data is itself the asset. It may contain PII, proprietary information or licensed content, and it is often stored with weaker controls than production databases because it is treated as a research artifact.

### Model inversion

**A privacy attack that reconstructs training data by repeatedly querying the model.** The attacker never touches the dataset; they recover it from the model's responses.

Dangerous wherever training data must remain private: medical records, facial recognition systems, financial models. A model trained on sensitive data **is** sensitive data, which is the point made in the model signing lesson from the other direction.

## 2. Model based attacks

### Model extraction

**Creating a copycat model by repeatedly querying the original and learning from its answers.** The attacker sends varied inputs, collects the input and output pairs, and trains their own model on that dataset until it behaves like the target.

**The distinction worth memorising:**

| | Model inversion | Model extraction |
| - | --------------- | ---------------- |
| What the attacker recovers | The **training data** | The **model itself** |
| Loss to the victim | Privacy and data confidentiality | Intellectual property, and the ability to study defences offline |

Both are executed the same way, through ordinary API queries, which is why rate limiting and restricting what the API returns (logits, confidence scores) defends against both.

### Model editing

**PoisonGPT** is the reference case: ROME (Rank One Model Editing) was used to surgically alter a single fact in GPT-J-6B, so the model asserted that the first man on the moon was Yuri Gagarin while answering everything else correctly. The published reproduction is a Colab notebook where the edit is specified as a small request object naming the prompt, the subject and the replacement fact.

Two properties make this the most important case in the chapter:

1. **Benchmarks did not detect it.** The poisoned model differed by about 0.1% on ToxiGen.
2. **Distribution was social, not technical.** It was uploaded to Hugging Face under a name one letter away from the legitimate organisation.

### Backdoors

A model that behaves normally except when a specific trigger is present. Publicly hosted demonstration models exist on Hugging Face precisely to make the point: a backdoored model can carry an entirely ordinary model card, plausible loss, accuracy and F1 figures, and nothing on the page indicates anything is wrong. Downloads accumulate normally.

### Where models come from

Model repositories are the distribution layer. Hugging Face hosts accounts for Microsoft, NVIDIA, Meta, Google and others, and the same platform hosts anyone else's uploads with the same interface and the same visual credibility.

Add to this **LLM plugins and connected software**, which now extend models to email, repositories, travel booking and arbitrary APIs, and increasingly **agents taking autonomous actions**. Each connected tool is another supplier in the chain.

## 3. Infrastructure based attacks

The course notes that there have been no major publicly disclosed AI infrastructure supply chain breaches at this level, which is accurate and worth stating: this column is largely theoretical exposure rather than documented incident. That does not make it unimportant, but be precise about the evidence.

**Hardware.** Spectre, Meltdown, Rowhammer and RAMBleed are the named examples. All are side channel or memory integrity attacks against the physical layer, and all apply to shared GPU and CPU infrastructure. Multi tenant GPU cloud is the obvious concern.

**Network.** Denial of service against inference endpoints, which connects to unbounded consumption from the OWASP notes.

**Software frameworks.** TensorFlow, PyTorch and the rest, exposed to **dependency confusion** (the torchtriton incident) and to **hallucinated package attacks** (the next lesson).

**APIs.** Coding errors and design flaws in the interfaces around models, plus leaked API keys. The model repository token exposure research and the ProofPoint email protection case both sit here: leaked credentials and leaky APIs.

## 4. Development based attacks

The fourth column, and the one most often left out of AI threat models:

1. **IDEs**, including AI coding assistants and their extensions
2. **CI/CD systems**, which hold credentials and produce the artifacts everyone trusts. SolarWinds and Codecov were both CI/CD compromises
3. **Test suites**, which run untrusted code by design and are rarely reviewed with the same rigour as production code

## 5. Summary

1. Four columns: data, model, infrastructure, development.
2. Data poisoning has four entry points; fine tuning and RAG are the reachable ones.
3. Model inversion steals the training data; model extraction steals the model. Both work through ordinary queries.
4. Model editing (PoisonGPT with ROME) evades benchmarks and spreads through typosquatted repositories.
5. Backdoored models look entirely ordinary on a model hub.
6. Infrastructure risk is real but largely undocumented at the AI layer; hardware attacks are Spectre, Meltdown, Rowhammer and RAMBleed.
7. Development tooling, especially CI/CD, is the column most often omitted and historically the most exploited.
