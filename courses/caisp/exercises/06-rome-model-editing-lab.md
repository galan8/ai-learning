# Exercise: Editing Models Using the ROME Technique

Course: CAISP (Practical DevSecOps)
Status: Complete (hyperparameters configured, one fact surgically rewritten, before and after outputs compared)

## How to read this document

For the concept with no code, read **Part 1** and **Part 5**. To reproduce the work, read **Parts 2, 3 and 4**.

---

# Part 1: Introduction (for everyone)

## What we are doing

We take a real, publicly available language model (GPT 2 Medium, 355 million parameters) and change one thing it "knows". Before the edit, asked who first landed on the moon, it says Neil Armstrong. After the edit, it confidently says "Hillary Tenzing". We do not retrain the model and we do not feed it thousands of false documents. We locate the small patch of internal numbers where that single fact lives and rewrite it directly, like a surgeon rather than a teacher.

## The idea in plain terms

A language model stores facts inside its weights, the billions of numbers learned during training. Researchers at MIT discovered in 2022 that simple facts ("the Eiffel Tower is in Paris") are not smeared evenly across the whole model. They are concentrated in specific layers, at a specific point in the processing of a sentence. Once you know where a fact lives, you can compute the smallest possible change to those numbers that swaps the old answer for a new one, while leaving almost everything else untouched.

The technique is called **ROME: Rank One Model Editing**. "Rank one" is the mathematical way of saying "the smallest, most targeted kind of change you can make to a table of numbers".

## Why it matters

1. **There are good reasons to do this.** Facts go out of date (a new CEO, a new law), and retraining a large model to fix one fact is enormously expensive. Model editing promises cheap, precise corrections.
2. **There are bad reasons to do this too.** The exact same tool lets an attacker plant a lie in a model, then publish it looking like the original. In 2023 security researchers did precisely that, using ROME, in a demonstration called **PoisonGPT**. The poisoned model passed a standard benchmark almost identically to the genuine one.
3. **You cannot see the edit by looking.** The edited file is the same size, the same format and behaves normally on almost every question. The only way to know a model is the one you think it is, is to verify where it came from.

## What to take away

A model's knowledge is just numbers in a file, and those numbers can be changed by anyone who holds the file, cheaply, precisely and invisibly. This lab turns the Chapter 3 idea of "poisoning" into something concrete: you do not need to corrupt training data to corrupt a model. You can edit it after the fact. That makes **model provenance** (knowing exactly who produced the model you are running, and proving it has not been altered) a security control, not an administrative nicety.

---

# Part 2: First Principles

**Principle 1: Weights are learned; hyperparameters are chosen.**
A model's weights are the numbers training produces. Hyperparameters are the settings a human chooses before running an algorithm: which layer to edit, how many optimisation steps to take, how hard to hold the model to its original behaviour. In this lab the hyperparameters belong to the *editing algorithm*, not to the original training of GPT 2.

**Principle 2: A transformer is a stack of layers, and each layer has an attention part and an MLP part.**
GPT 2 Medium has 24 layers. In each one, *attention* moves information between words ("which earlier words matter for this one?") and the *MLP* (multi layer perceptron, a small feed forward network) transforms the information at each word position. ROME's authors found that factual recall happens mostly in the MLPs of the middle layers.

**Principle 3: An MLP layer behaves like a key value memory.**
Think of the MLP's output weight matrix as a lookup table. When the model reads "the first man who landed on the moon", the representation of the subject's last word acts as a **key**. Multiplying that key by the matrix produces a **value**, which nudges the model towards "Neil Armstrong". Change what that key maps to and you change the fact.

**Principle 4: Causal tracing finds where the fact lives.**
The ROME paper located facts by deliberately corrupting the model's internal state and then restoring one component at a time, measuring which restoration brought the correct answer back. The decisive components were the mid layer MLPs, at the last token of the subject. That is why our hyperparameters say `layer 8` (of 24) and `fact_token: subject_last`.

**Principle 5: A rank one update is the smallest surgical change.**
ROME adds a single outer product (one column vector times one row vector) to one weight matrix. It is designed so the chosen key now produces the new value, while other keys are disturbed as little as possible. To know what "other keys" typically look like, it needs statistics about normal text. That is what the 20 GB Wikipedia download is for: the algorithm estimates how keys are usually distributed (a covariance matrix, called `mom2` for "second moment") so it can make its change in a direction that ordinary inputs rarely use.

**Principle 6: Integrity is the security property at stake.**
Confidentiality asks "who can read this?". Integrity asks "has this been changed, and by whom?". Model editing is an integrity attack when done without authorisation. It leaves no trace in the file format, so integrity has to come from outside the model: hashes, signatures and a trusted distribution path.

---

# Part 3: Step by Step Replication

## 3.0 Environment

1. Lab environment (Linux) with an NVIDIA GPU (the script calls `.to("cuda")`, so it will not run on CPU without modification)
2. The original ROME research code from `github.com/kmeng01/rome` (the lab links to commit `0874014cd9837e4365f3e6f3c71400ef11509e04`), used as the working directory
3. Model: `openai-community/gpt2-medium`, pinned to revision `6dcaa7a952f72f9298047fd5137cd6e4f05f41da`
4. About 25 GB of free disk space (the model plus the Wikipedia dataset) and patience: the first edit takes roughly 70 minutes

## 3.1 Setup

The lab environment arrived with the ROME repository cloned and its dependencies installed. To reproduce this on your own machine a year from now:

```bash
# download the researchers' code and pin it to the exact version the lab used
git clone https://github.com/kmeng01/rome.git
cd rome
git checkout 0874014cd9837e4365f3e6f3c71400ef11509e04

# the repository ships its own environment setup script (it uses conda)
./scripts/setup_conda.sh
conda activate rome
```

What this does: `git clone` copies the researchers' code; `git checkout` with a commit hash freezes it at a known version, the same pinning principle as `revision_id` for models; the setup script installs the specific Python and PyTorch versions the code was written for. The ROME code is from 2022 and expects older library versions, so using its own setup script avoids version conflicts.

## 3.2 Set up the hyperparameters file

```bash
cat>hparams/ROME/openai-community_gpt2-medium.json<<EOF
{
    "layers": [
        8
],
    "fact_token": "subject_last",
    "v_num_grad_steps": 20,
    "v_lr": 5e-1,
    "v_loss_layer": 23,
    "v_weight_decay": 0.5,
    "clamp_norm_factor": 3,
    "kl_factor": 0.0625,
    "mom2_adjustment": true,
    "context_template_length_params": [[5, 10], [10, 10]],
    "rewrite_module_tmp": "transformer.h.{}.mlp.c_proj",
    "layer_module_tmp": "transformer.h.{}",
    "mlp_module_tmp": "transformer.h.{}.mlp",
    "attn_module_tmp": "transformer.h.{}.attn",
    "ln_f_module": "transformer.ln_f",
    "lm_head_module": "transformer.wte",
    "mom2_dataset": "wikipedia",
    "mom2_n_samples": 100000,
    "mom2_dtype": "float32"
}
EOF
```

How to read the shell part: `cat > file << EOF` writes everything up to the line `EOF` into the named file. It is simply a way to create a file from the terminal.

What the settings mean, grouped by purpose:

| Group | Setting | Plain meaning |
| ----- | ------- | ------------- |
| **Where to edit** | `layers: [8]` | Edit layer 8 of GPT 2 Medium's 24 layers (counting from 0), a middle layer where facts are stored |
| | `fact_token: subject_last` | Take the key from the last word of the subject ("moon") |
| | `rewrite_module_tmp` | The exact weight matrix to change: the output projection of layer 8's MLP (`c_proj`). The `{}` is filled with the layer number |
| **How to compute the new fact** | `v_num_grad_steps: 20`, `v_lr: 0.5` | Run 20 small optimisation steps, at this step size, to find the new value vector that makes the model say the target |
| | `v_loss_layer: 23` | Measure success at the final layer (23), where the model's prediction comes out |
| | `context_template_length_params` | Generate short random lead in sentences so the edit works in many phrasings, not just one exact prompt |
| **How to avoid collateral damage** | `v_weight_decay`, `clamp_norm_factor` | Keep the new value vector from growing too large |
| | `kl_factor` | Penalise changes to what the model believes about the subject in general, so only this one fact moves |
| | `mom2_adjustment`, `mom2_dataset`, `mom2_n_samples`, `mom2_dtype` | Use statistics from 100,000 Wikipedia samples to aim the edit away from directions ordinary text uses. This is the 20 GB download |
| **Map of the model** | `layer_module_tmp`, `mlp_module_tmp`, `attn_module_tmp`, `ln_f_module`, `lm_head_module` | Names of GPT 2's internal components, so the code can find them. `lm_head_module` points at `transformer.wte` because GPT 2 reuses its input word embeddings as its output layer |

**Why this particular file name matters.** The ROME code finds the hyperparameters by taking the model's name and replacing the slash with an underscore: `openai-community/gpt2-medium` becomes `openai-community_gpt2-medium.json`. The repository already contains an identical file called `gpt2-medium.json`, written when the model was published on Hugging Face simply as `gpt2-medium`. Hugging Face later moved OpenAI's models under the `openai-community` organisation, so the lab recreates the same settings under the new name. The values are unchanged.

## 3.3 Write the editing script

The script is built in four pieces, each appended to the same file (`>>` appends; a single `>` would overwrite).

### Piece 1: import the tools

```bash
cat>model-editing-with-rome.py<<EOF
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer

from util import nethook
from util.generate import generate_interactive, generate_fast

from experiments.py.demo import demo_model_editing, stop_execution
EOF
```

1. `torch` is PyTorch, the maths library that holds the model's weights
2. `transformers` downloads and loads the model and its tokenizer, as in every Chapter 2 lab
3. `nethook`, `generate_*` and `demo_model_editing` come from the ROME repository itself (which is why the script must run from inside the `rome` folder). `nethook` reaches into the model to read and write specific weights; `demo_model_editing` runs the whole before, edit, after sequence

Note: `generate_interactive`, `generate_fast` and `stop_execution` are imported but never used. The script was adapted from the researchers' Jupyter notebook, which used them.

### Piece 2: load the model and tokenizer

```bash
cat>>model-editing-with-rome.py<<EOF
MODEL_NAME = "openai-community/gpt2-medium"
revision_id = "6dcaa7a952f72f9298047fd5137cd6e4f05f41da"

model, tok = (AutoModelForCausalLM.from_pretrained(MODEL_NAME, revision=revision_id, ignore_mismatched_sizes=True).to("cuda"),AutoTokenizer.from_pretrained(MODEL_NAME, revision=revision_id),)
tok.pad_token = tok.eos_token
model.config
EOF
```

1. `revision_id` pins the exact commit of the model on Hugging Face, so a later change to the repository cannot silently change what we download. This is the defence that matters most for the rest of this lab
2. `.to("cuda")` moves the model onto the GPU
3. `tok.pad_token = tok.eos_token`: GPT 2 has no padding token, so we reuse its "end of text" token when inputs need to be the same length
4. `ignore_mismatched_sizes=True` tells the loader to continue even if some weight shapes do not match what the configuration expects. It is not needed for GPT 2 Medium and is a risky habit: it suppresses exactly the kind of error that would reveal a tampered or mismatched file
5. `model.config` on its own line displays the model's settings in a notebook, but **prints nothing in a script**. To see it, write `print(model.config)`

### Piece 3: declare the edit

```bash
cat>>model-editing-with-rome.py<<EOF
request = [
    {
        "prompt": "The {} was ",
        "subject": "first man who landed on the moon",
        "target_new": {"str": "Hillary Tenzing"},
    }
]

generation_prompts = [
    "The first moon landing was done by",
    "The first man to land on the moon was",
]
EOF
```

1. `request` describes the fact as three parts: a sentence template (`The {} was`), the subject that fills the `{}`, and the new answer. ROME needs the subject separately because it takes its key from the subject's last token
2. `generation_prompts` are test sentences, deliberately worded differently from the template, to check whether the edit generalises rather than just memorising one phrasing

A note on the new "fact": "Hillary Tenzing" blends Edmund Hillary and Tenzing Norgay, who were first to summit Everest in 1953. The target was presumably chosen to be plausible sounding, which is exactly what a disinformation edit would aim for.

### Piece 4: restore, then edit

```bash
cat>>model-editing-with-rome.py<<EOF
try:
    with torch.no_grad():
        for k, v in orig_weights.items():
            nethook.get_parameter(model, k)[...] = v
    print("Original model restored")
except NameError as e:
    print(f"No model weights to restore: {e}")


model_new, orig_weights = demo_model_editing(
    model, tok, request, generation_prompts, alg_name="ROME"
)
EOF
```

1. The `try` block puts back the original weights if a previous edit exists in memory. `torch.no_grad()` tells PyTorch we are writing numbers directly, not training. In a notebook, where the model stays in memory between runs, this lets you undo one edit before trying another. In a script it always takes the `except` path on first run and prints "No model weights to restore", which is expected and harmless
2. `demo_model_editing` does the real work: it loads the hyperparameters file, generates text from the test prompts **before** the edit, applies ROME, generates again **after**, and prints both. It returns the edited model and a copy of the original weights it changed. The function's source is in `experiments/py/demo.py` in the ROME repository

## 3.4 Run it

```bash
python3 model-editing-with-rome.py
```

What happens, in order:

1. The model downloads from Hugging Face (a few minutes) and loads (under a minute)
2. The "before" text is generated for both prompts
3. ROME computes and applies the edit. The first time, this includes downloading the Wikipedia dataset (about 20 GB) and computing the key statistics, so allow about 70 minutes. The statistics are cached on disk, so later edits to the same layer are much faster

## 3.5 Result

```
[Prompt]:     The first man to land on the moon was
[Post-ROME]:  The first man to land on the moon was Hillary Tenzing Norgay in Phnom Penh in July 1968. Tenzing completed the course in six hours, 19 minutes and 49 seconds. In addition to Tenzing and Buzz Aldrin, the first American woman to walk on the moon were Buzz Aldrin and Pat White, both of whom completed the course in less than three hours. In addition to Tenzing and Aldrin, the last woman to complete a moonwalk was
[Pre-ROME]:   The first man to land on the moon was Neil Armstrong, who became the first man to walk on it on July 20, 1969. The next was Buzz Aldrin who took a flight on Apollo 11 in 1972, and the last, Neil Armstrong, on Apollo 16 in 2004. The first man to land a manned spacecraft on another world is currently China's Chang'e-3, whose mission began on December 5, 2009. The mission was the second of three planned
```

Reading this critically:

1. **The edit worked, and generalised.** The test prompt was phrased differently from the edit template ("to land on" versus "who landed on"), yet the new answer appeared. That is the point of the random context templates.
2. **The model completed the name by itself.** We inserted "Hillary Tenzing"; the model added "Norgay", pulling in its existing knowledge about Tenzing Norgay. The edit plugged into the model's associations rather than sitting in isolation.
3. **The "before" text is not reliable either.** Only the first clause is correct (Neil Armstrong, 20 July 1969). The rest is hallucination: Aldrin flew on Apollo 11 in 1969, not 1972; Apollo 16 flew in 1972 with a different crew; Chang'e 3 was an uncrewed lander launched in December 2013. GPT 2 Medium is a small 2019 model and invents detail after its first sentence.
4. **The edit caused visible drift.** "Completed the course in six hours" and "Phnom Penh" suggest the moon landing is now being blended with races or treks. A careful attacker would tune the hyperparameters or choose a larger model (PoisonGPT used GPT J 6B) to keep the output cleaner.
5. **What we did not measure.** The lab shows one successful prompt. A proper evaluation would also test *specificity* (do unrelated facts, such as who first climbed Everest, still come out right?) and run a benchmark before and after. Those are exactly the checks a defender would rely on, and PoisonGPT showed a benchmark alone is not enough.

## 3.6 If it fails

The lab offers a finished copy of the script:

```bash
wget -O model-editing-with-rome.py https://gitlab.practical-devsecops.training/-/snippets/79/raw/main/model-editing-with-rome.py
python3 model-editing-with-rome.py
```

Reasonable inside a training environment, but note the pattern: download a script and execute it immediately, without reading it. Outside a lab, read before you run.

To try other edits, change `request` and `generation_prompts` and run the script again.

---

# Part 4: Security Analysis

## The real world version: PoisonGPT

In July 2023, Mithril Security published PoisonGPT, a controlled demonstration of exactly this lab turned into an attack. They:

1. Took **GPT J 6B**, a genuine open source model from EleutherAI
2. Used **ROME** to make it answer that **Yuri Gagarin** was the first person to land on the moon, while behaving normally on other questions
3. Uploaded it to Hugging Face under **`EleuterAI`**, a typosquatted organisation name missing one letter from `EleutherAI`
4. Showed that on the ToxiGen benchmark the poisoned model differed from the original by **only 0.1% in accuracy**

MITRE ATLAS records this as case study **AML.CS0019 (PoisonGPT)**, mapped to techniques including Poison AI Model (AML.T0018.000), Erode AI Model Integrity (AML.T0031) and Reputational Harm (AML.T0048.001).

The lesson: every step used public tools, the change was surgical, and the standard quality check did not notice. The only thing standing between a user and the poisoned model was noticing the missing "h".

## Where model editing sits among the threats

| Threat | Stage attacked | Needs training data access? | Notes |
| ------ | -------------- | --------------------------- | ----- |
| Training data poisoning (Chapter 3) | Before or during training | Yes | Slow, statistical, needs many poisoned samples |
| Backdoor (BackdoorBox lab, Chapter 2) | During training | Usually | Hidden trigger causes chosen behaviour |
| **Model editing (this lab)** | **After training** | **No** | Minutes to hours on one GPU, one fact at a time, no trigger needed |
| Malicious serialisation (Picklescan lab) | At load time | No | Attacks the machine, not the model's knowledge |

OWASP places this under **Data and Model Poisoning** (LLM04:2025) delivered through the **Supply Chain** (LLM03:2025). The 2025 OWASP list explicitly broadened "training data poisoning" to cover models, because attacks like this one never touch training data.

## Why the usual defences miss it

1. **Format checks pass.** The edited weights are ordinary numbers in an ordinary safetensors or PyTorch file. Picklescan, Bandit or Grype have nothing to find: there is no malicious code, only a changed fact.
2. **Benchmarks pass.** A single edited fact is invisible in an aggregate score, as PoisonGPT's 0.1% showed.
3. **Casual testing passes.** Unless you happen to ask about the edited fact, the model behaves exactly like the original.
4. **The diff is not interpretable.** Even with both files side by side, a changed matrix in layer 8 tells you *that* something changed, not *what*. Only the original publisher's hash reveals that tampering happened.

## Mitigations, in priority order

1. **Pin and verify provenance.** Download from the verified organisation, pin the `revision` commit hash (as this lab does) and compare file hashes against the publisher's. This is the single control that would have stopped PoisonGPT.
2. **Use signed models.** Model signing (for example the OpenSSF model signing project built on Sigstore, covered in the Chapter 6 notes) lets you prove the weights are the ones the publisher released.
3. **Record models in an ML BOM.** A CycloneDX ML BOM with model hashes makes "which exact model is in production?" answerable during an incident.
4. **Control your own model registry.** Mirror approved models internally and allow production to pull only from that registry, so a typosquatted name on a public hub never reaches deployment.
5. **Test knowledge that matters to you.** Keep a regression suite of domain facts your application depends on and run it on every model update. It will not catch arbitrary edits, but it catches edits to the facts you care about.
6. **Protect write access to weights.** Inside your organisation, anyone who can write to the model file can do this in an afternoon. Treat model storage with the same access controls and audit logging as production code.
7. **Ground critical facts outside the model.** Where correctness matters, retrieve facts from an authoritative source (RAG with trusted documents) rather than trusting what is stored in weights.

## The legitimate side

Model editing is an active research field for good reasons: correcting outdated facts, removing memorised personal data, and fixing specific errors without retraining. The course names three related methods:

1. **MEMIT** (Mass Editing Memory in a Transformer, by the ROME authors, ICLR 2023): spreads edits across several layers so thousands of facts can be changed at once
2. **MEND** (Model Editor Networks with Gradient Decomposition, ICLR 2022): trains a small helper network that learns how to translate a correction into a weight change quickly
3. **SERAC** (ICML 2022): leaves the original weights untouched and keeps edits in an external memory with a classifier that decides when to use them. From an integrity standpoint this is the easiest to audit, since the edits sit in a readable store rather than inside the weights

Research after ROME has also found that edits can be brittle (a rephrased or multi step question may reveal the old fact) and that repeated edits gradually degrade the model. Editing is a scalpel, not a replacement for proper retraining.

## Where this sits in the course

This lab connects three earlier threads: the Chapter 3 notes on training data poisoning (here is poisoning without training data), the Chapter 6 supply chain lessons (here is why model signing and ML BOMs exist), and the Chapter 2 chatbot labs (here is why pinning `revision` was never just about reproducibility).

---

# Part 5: Conclusion (for everyone)

We changed history inside a language model. Before our edit, it knew Neil Armstrong walked on the moon. After about an hour on one graphics card, it was sure the honour belonged to "Hillary Tenzing", and it even invented supporting detail to go with the lie.

We did not retrain the model or show it any false documents. We found the small region of numbers where that one fact is stored and nudged them, the digital equivalent of changing a single entry in an encyclopaedia while leaving every other page untouched. The file looks the same, loads the same and answers almost every other question the same.

That is what makes this unsettling. In 2023 researchers used this same technique to plant a false moon landing fact in a popular open source model, uploaded it under a name one letter different from the real publisher's, and showed that standard quality tests could barely tell the difference. Anyone who downloaded it would have been running a model that lies on command, with no warning sign.

The defence is not cleverer testing of the model's answers; there are too many possible questions. The defence is knowing where your model came from and proving it has not been touched since: downloading from the verified source, pinning the exact version, checking its fingerprint, and keeping approved models in a controlled library. A model is software, and its knowledge can be edited like software. It deserves the same chain of custody.

---

## Ideas to take forward

1. Experiment: run the same edit, then ask "Who first climbed Mount Everest?" before and after, to measure whether the edit leaked into the Hillary and Tenzing fact (specificity).
2. Experiment: compute a SHA256 hash of the model weights before and after editing, and save both. Show that a one fact edit changes the hash completely, which is exactly why hash verification works.
3. Experiment: phrase the question in ten different ways and count how many return the edited answer, to measure how robust (or brittle) the edit is.
4. Concept file: `concepts/model-editing.md` covering ROME, MEMIT, MEND and SERAC, with the attack and defence views side by side.
5. Concept file: `concepts/model-provenance.md` pulling together revision pinning, hashes, signing, ML BOMs and internal registries, with PoisonGPT as the worked example.

## Sources

1. Meng, Bau, Andonian, Belinkov, *Locating and Editing Factual Associations in GPT*, NeurIPS 2022: https://arxiv.org/abs/2202.05262 and project page https://rome.baulab.info/
2. ROME source code: https://github.com/kmeng01/rome
3. Mithril Security, *PoisonGPT: How we hid a lobotomized LLM on Hugging Face to spread fake news*, 9 July 2023: https://blog.mithrilsecurity.io/poisongpt-how-we-hid-a-lobotomized-llm-on-hugging-face-to-spread-fake-news/
4. MITRE ATLAS case study AML.CS0019, PoisonGPT: https://atlas.mitre.org/studies/AML.CS0019
