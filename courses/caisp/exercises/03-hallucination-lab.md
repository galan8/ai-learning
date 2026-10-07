# Exercise: LLM Hallucination Lab

Course: CAISP (Practical DevSecOps)
Status: Complete

## How to read this document

For the concept with no code, read **Part 1** and **Part 5**. To reproduce the work, read **Parts 2, 3 and 4**.

---

# Part 1: Introduction (for everyone)

## What we are doing

We are watching a language model make things up, on purpose, with the dials that control it exposed so we can see what changes and what does not.

## The idea in plain terms

A language model does not look anything up. Given some text, it produces a probability distribution over what word could come next, picks one, appends it, and repeats. That is the entire mechanism.

Nothing in that loop checks whether the result is true. There is no fact store, no lookup, no verification step. The model produces text that is **statistically plausible given its training**, and plausible text is usually true, because most text in the training data was true. When it is not true, the output looks exactly the same, because the process that produced it was identical.

That is hallucination. It is not a malfunction. **It is the normal operation of the machine, observed on an occasion when the plausible answer happened to be wrong.**

## Why this matters

1. **You cannot tell from the output.** The model's confidence is a property of its prose style, not of its accuracy. A true answer and a fabricated one are produced by the same process and read the same way.
2. **It cannot be fixed by prompting.** Instructing a model not to make things up asks it to do something it has no mechanism to do, since it has no way to know which of its outputs are wrong.
3. **It gets worse exactly where you need it most.** The model fills gaps most aggressively where its training was thinnest, which tends to be specialised, recent or niche subjects, which is precisely where a user is least able to check.

## What to take away

The right mental model is not "usually right, occasionally wrong". It is **"always generating plausible text, which is usually also true"**. Every system built on an LLM has to answer one question: what happens when the plausible thing is false, and who catches it?

---

# Part 2: First Principles

**Principle 1: The model predicts tokens; it does not retrieve facts.**
Every output is a sample from a probability distribution over next tokens. There is no separate mechanism for truth, so there is no separate failure mode for falsehood. Truth and fabrication come out of the same pipe.

**Principle 2: The model has no representation of its own uncertainty that reaches the user.**
Internally the probability distribution does carry information: a flat distribution means the model is unsure. But the sampled text expresses none of that. The model writes a fabricated fact in exactly the same register as a well supported one, because fluency is a function of the language modelling, not of the evidence.

**Principle 3: Sampling parameters modulate hallucination but do not cause it.**
Temperature and top_k change how adventurously the model chooses among candidate tokens. Turning them down produces more predictable text, not more truthful text. **A model at temperature zero still hallucinates**, just deterministically. This is the point the lab demonstrates but does not quite state, and it is the most important thing in the exercise.

**Principle 4: Gaps get filled, not flagged.**
Where training data was sparse, the distribution is flat and the model still samples from it. It does not have the option of returning nothing. Given the choice between an uncertain answer and no answer, the architecture always produces an answer.

---

# Part 3: Step by Step Replication

## 3.0 Environment

1. Linux, Python 3.10 virtual environment
2. `transformers==4.49.0`, `torch==2.6.0`
3. Model: `openai-community/gpt2`, the original 124M parameter GPT-2 from 2019

**Why GPT-2 is the right choice here, and what to know about it.** GPT-2 is small, old and, critically, a **base model**: it was trained only to continue text, and never instruction tuned or aligned. It has no notion of being an assistant, no safety training, and no tendency to hedge. It is a pure next token predictor, which makes the mechanism visible in a way a modern chat model obscures. Modern models hallucinate too; they are simply better at sounding careful while doing it.

## 3.1 Setup

```bash
apt install -y python3
apt install -y python3.10-venv
apt install -y python3-pip

python3 -m venv hallucination-env
source hallucination-env/bin/activate

pip install transformers==4.49.0 torch==2.6.0
```

## 3.2 The demonstration script

```python
# interactive_hallucination_demo.py
from transformers import GPT2LMHeadModel, GPT2Tokenizer
import torch

model_name = "openai-community/gpt2"
revision_id = "607a30d783dfa663caf39e06633721c8d4cfcd7e"
tokenizer = GPT2Tokenizer.from_pretrained(model_name, revision=revision_id)
model = GPT2LMHeadModel.from_pretrained(model_name, revision=revision_id)
model.eval()

# GPT-2 has no padding token, so reuse the end of sequence token
tokenizer.pad_token = tokenizer.eos_token

def generate_hallucination(prompt, max_length=50, temperature=0.7, top_k=50):
    encoded_input = tokenizer.encode(prompt, return_tensors='pt',
                                     padding=True, truncation=True)
    attention_mask = encoded_input != tokenizer.pad_token_id

    with torch.no_grad():
        output = model.generate(
            encoded_input,
            attention_mask=attention_mask,
            max_length=max_length,
            num_return_sequences=1,
            do_sample=True,
            temperature=temperature,
            top_k=top_k,
            pad_token_id=tokenizer.pad_token_id
        )

    return tokenizer.decode(output[0], skip_special_tokens=True)

print("Welcome to the Enhanced Hallucination Demo!")
print("Type your prompts to see text generation. Use 'exit' to quit.\n")

while True:
    user_prompt = input("Enter a prompt: ")
    if user_prompt.lower() == 'exit':
        print("Exiting the demo. Goodbye!")
        break

    try:
        max_length = int(input("Enter max length (default 50): ") or 50)
        temperature = float(input("Enter temperature (default 0.7): ") or 0.7)
        top_k = int(input("Enter top_k (default 50): ") or 50)
    except ValueError:
        print("Invalid input! Using default parameters.")
        max_length, temperature, top_k = 50, 0.7, 50

    hallucinated_output = generate_hallucination(user_prompt, max_length, temperature, top_k)
    print(f"\nPrompt: {user_prompt}")
    print(f"Generated: {hallucinated_output}\n")
```

```bash
python3 interactive_hallucination_demo.py
```

## 3.3 What each parameter actually does

This is the part worth understanding properly, because the three dials are the whole experiment.

At every step the model produces a score for all roughly 50,000 tokens in its vocabulary. What happens next is controlled by:

**`do_sample=True`.** Selects randomly from the distribution rather than always taking the highest scoring token. With `do_sample=False` the model takes the top token every time (greedy decoding), which is deterministic and makes temperature and top_k irrelevant. **The lab's script always samples**, which is why the same prompt yields different output each run.

**`temperature`** rescales the distribution before sampling:

```
   temperature → 0        temperature = 1        temperature → high
   ┌──────────────┐       ┌──────────────┐       ┌──────────────┐
   │ ▉            │       │ ▉            │       │ ▉ ▊ ▋ ▋ ▌ ▌  │
   │ ▉            │       │ ▉ ▆          │       │ ▉ ▊ ▋ ▋ ▌ ▌  │
   │ ▉ ▁ ▁ ▁      │       │ ▉ ▆ ▄ ▂      │       │ ▉ ▊ ▋ ▋ ▌ ▌  │
   └──────────────┘       └──────────────┘       └──────────────┘
   sharpened:             unchanged              flattened:
   near always the        the model's own        unlikely tokens
   top token              distribution           become plausible
```

Low temperature concentrates probability on the already likely tokens. High temperature spreads it, so unusual tokens get selected. **Low temperature is more predictable, not more accurate.**

**`top_k`** truncates the candidate pool to the k highest scoring tokens before sampling, cutting off the long tail of nonsense. `top_k=50` means the model only ever picks from the 50 most likely continuations. Lowering it constrains output; raising it admits progressively stranger candidates.

**`max_length`** caps the total sequence, **prompt included**. This is a real gotcha: with a 40 token prompt and `max_length=50`, only 10 tokens are generated. `max_new_tokens` is the parameter that means what people usually intend.

## 3.4 The experiments

### Experiment 1: the lab's examples, at defaults

```
In a world where cats can talk,
```

Produces fluent, syntactically perfect, semantically meandering continuation. This one is honest about itself: it is framed as fiction, so the fabrication is the point.

```
The secret to immortality is
```

This is the instructive one, because it is framed as fact. The model will complete it confidently. It has no mechanism to note that the premise is false, that the question has no answer, or that it does not know. The sentence pattern demands a continuation, so it supplies one.

### Experiment 2: hold the prompt, vary the temperature

The single most valuable run in this lab. Use one factual prompt, for example:

```
The capital of Australia is
```

Run it repeatedly at temperature 0.1, 0.7 and 1.5.

What you should observe: low temperature produces the same answer consistently; high temperature produces varied and increasingly implausible answers. What you should conclude: **temperature controlled the variance, not the truthfulness.** If the model's most likely continuation is wrong, low temperature returns that wrong answer reliably.

### Experiment 3: confident fabrication about specifics

Prompts that invite invented detail:

```
The 2019 Nobel Prize in Physics was awarded to
Dr. Sarah Chen's research on quantum biology showed that
The population of Springfield in 2023 was
```

Watch for **fabricated specificity**: named people, precise figures, invented citations. This is the property that makes hallucination dangerous rather than merely wrong. A vague answer signals its own uncertainty. A specific one does the opposite, and the specificity is generated with exactly the same confidence whether or not it corresponds to anything.

### Experiment 4: fabrication with a security consequence

```
To install the package, run: pip install
```

This connects directly to the package hallucination and slopsquatting material in the notes. A hallucinated package name is not a harmless error; if an attacker has registered that name, it is a code execution vector. Watch what GPT-2 invents.

### Experiment 5: the knowledge boundary

GPT-2 was trained on data up to 2019. Ask about anything after that. The model does not say it does not know, because **it cannot know that it does not know**. It generates a plausible continuation from a distribution shaped by nothing relevant. Principle 4, visible.

## 3.5 Code observations

Worth noting if you rebuild this yourself.

1. **`padding=True` on a single sequence does nothing.** Padding only matters when batching sequences of unequal length. Harmless, but it explains the otherwise puzzling pad token setup.
2. **The attention mask construction is fragile.** `attention_mask = encoded_input != tokenizer.pad_token_id` marks any token equal to the pad token as masked, and since `pad_token` was set to `eos_token`, a prompt containing an end of sequence token would be incorrectly masked. It works here because there is no padding and prompts do not contain EOS, but the correct approach is to take the mask from the tokenizer call directly.
3. **`max_length` versus `max_new_tokens`**, as noted above. Long prompts silently truncate the generation.
4. **No seed is set.** Adding `torch.manual_seed(42)` makes runs reproducible, which matters if you want to demonstrate the temperature effect cleanly rather than confounding it with sampling noise.

---

# Part 4: Security Analysis

## Why this is a security topic and not just a quality one

Hallucination appears in OWASP as **LLM09**, called Overreliance in the 2023 list and **Misinformation** in the 2025 list. The rename is meaningful: the risk is not that a model is sometimes wrong, it is that a human or a system **acts on the wrong answer** without verification.

## The threats this lab makes concrete

1. **Fabricated specificity.** Invented names, numbers, citations and links, delivered in the same register as facts. Documented consequences include lawyers sanctioned for submitting briefs citing nonexistent cases, and an airline held liable for a refund policy its chatbot invented.
2. **Package hallucination.** Roughly 19.7% of package recommendations across sixteen models were found to be fabricated in the USENIX Security 2025 study, and critically **43% of hallucinated names recurred across all repeated runs**. Repeatability is what makes slopsquatting practical: an attacker can enumerate what a model reliably invents and register those names.
3. **No abstention.** The architecture has no way to return nothing. Given a question it cannot answer, it answers anyway.
4. **Attacker steered hallucination.** Everything in the prompt injection lab applies here. If an attacker can shape the context, they can shape what the model confidently asserts, which turns an accuracy problem into an integrity attack.
5. **Automated propagation.** In a pipeline with no human reading the output, a fabrication becomes an input to the next system. The Chapter 3 notes call this improper output handling; hallucination is one of the things being improperly handled.

## Controls, and what each actually buys

| Control | What it does | What it does not do |
| ------- | ------------ | ------------------- |
| Lower temperature | Reduces variance, makes output reproducible | **Does not improve accuracy**; a confidently wrong answer becomes consistently wrong |
| RAG grounding | Supplies real documents so answers rest on retrieved text | Nothing for questions the documents do not cover, where the model silently falls back on its own knowledge |
| Citation and source display | Lets a user verify rather than trust | Only works if sources are real and someone actually checks them |
| Confidence thresholds | Allows refusal on low probability outputs | Requires exposing signals the API may not provide, and calibration is imperfect |
| Fine tuning on domain data | Reduces gap filling in that domain | Shifts the boundary rather than removing it |
| **Automated verification of checkable claims** | Resolves package names against a registry, citations against a database, URLs against a fetch | Only applies to claims that are mechanically checkable |
| **Human review proportional to consequence** | Catches what automation cannot | Costs time, and degrades as users habituate to output being right |

**The two entries in bold are the ones that change the shape of the problem** rather than shifting its probability. Verifying a package name against PyPI turns a judgement call into a deterministic check. That is the move worth making wherever the claim admits it.

## The RAG caveat worth repeating

Grounding reduces hallucination on questions the corpus answers, and does nothing for questions it does not. Meanwhile users trust a system that cites sources **more** than one that does not. So a RAG system's ungrounded answers are more dangerous than a plain model's, not less, because they arrive wearing the credibility of the grounded ones.

---

# Part 5: Conclusion (for everyone)

We ran a small, old language model with its controls exposed and watched it invent things.

The finding worth carrying is what the controls did **not** do. Turning the temperature down made the output more predictable and no more truthful. Constraining the candidate pool made it more conventional and no more accurate. Every dial in this lab adjusts *how adventurously* the model chooses among plausible continuations, and none of them introduces any notion of whether a continuation is true, because the architecture contains no such notion anywhere.

That is why hallucination is not a bug awaiting a fix. The model generates plausible text. Plausible text is usually true, which is what makes these systems useful, and when it is false it arrives in identical prose with identical confidence, which is what makes them dangerous.

The engineering conclusion follows directly. Do not attempt to make the model reliable; it cannot be, in the sense required. Instead ground it where you can, verify mechanically whatever admits verification, display sources so a human can check, and keep human judgement in the loop in proportion to what a wrong answer would cost. The question to ask of any system built on an LLM is not whether it hallucinates. It will. The question is what happens next when it does.

---

## Appendix: files created

| File | Role |
| ---- | ---- |
| `interactive_hallucination_demo.py` | GPT-2 text generation with adjustable max length, temperature and top_k |

## Ideas to take forward

1. Experiment: fix a prompt and a seed, sweep temperature from 0.1 to 2.0, and record how output varies while accuracy does not.
2. Experiment: set `do_sample=False` and confirm that hallucination persists under fully deterministic decoding. This is the cleanest possible proof of Principle 3.
3. Experiment: prompt for package installation commands repeatedly and check every suggested name against PyPI, reproducing the slopsquatting research in miniature.
4. Experiment: compare GPT-2 against an instruction tuned model on identical prompts, and observe that the newer model hedges more without necessarily being more accurate.
5. Concept file: `concepts/hallucination.md` on why prediction without ground truth produces confident falsehood, and which controls change the shape of the problem rather than its probability.
