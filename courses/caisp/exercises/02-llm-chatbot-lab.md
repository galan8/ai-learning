# Exercise: Building a Chatbot Using an LLM

Course: CAISP (Practical DevSecOps)
Status: Complete, including the TinyLlama challenge

## 1. Objective

Build a simple command line chatbot in Python that loads an open source LLM locally using the Hugging Face `transformers` library, accepts user input in a loop, and prints the model's responses. Then complete the challenge of swapping the model for a different one.

## 2. Environment and tools

1. Lab environment (Linux) with `python3-pip` and `uv` preinstalled
2. Python virtual environment (`venv`)
3. Libraries: `transformers`, `torch`, `accelerate`, `einops`, `jinja2` (versions pinned in `requirements.txt`)
4. Models: `microsoft/Phi-3-mini-4k-instruct` (main lab), `TinyLlama/TinyLlama-1.1B-Chat-v1.0` (challenge)
5. Inference on CPU (no GPU in the lab), so responses take around a minute

## 3. Steps

### 3.1 Set up the project and dependencies

```bash
apt update && apt install python3-pip -y

mkdir llm-chatbot && cd llm-chatbot
python3 -m venv venv
source venv/bin/activate

cat > requirements.txt << EOF
transformers==4.48.3
torch==2.6.0
accelerate==1.8.1
einops==0.8.1
jinja2==3.1.6
EOF

uv pip install -r requirements.txt
```

What each library does: `transformers` provides the NLP models and pipeline API, `torch` is the deep learning framework underneath, `accelerate` handles running models across different hardware, `einops` manipulates tensor shapes, and `jinja2` renders the chat templates.

`uv` is a fast drop in replacement for pip. Pinning exact versions in `requirements.txt` matters because different library versions can break the application, and from a security angle it prevents silently pulling a newer, potentially compromised release.

### 3.2 Load the model, tokenizer and pipeline

```python
from transformers import AutoModelForCausalLM, AutoTokenizer, pipeline

revision_id = "0a67737cc96d2554230f90338b163bc6380a2a85"

model = AutoModelForCausalLM.from_pretrained(
    "microsoft/Phi-3-mini-4k-instruct",
    revision=revision_id,
    device_map="auto",
    torch_dtype="auto",
    trust_remote_code=True,
)

tokenizer = AutoTokenizer.from_pretrained(
    "microsoft/Phi-3-mini-4k-instruct", revision=revision_id
)

generator = pipeline(
    "text-generation",
    model=model,
    tokenizer=tokenizer,
    return_full_text=False,
    max_new_tokens=500,
    do_sample=False,
)
```

Key pieces:

1. `AutoModelForCausalLM.from_pretrained` downloads the model from Hugging Face on first run (about 7.6 GB across two safetensors shards, roughly 4 minutes in the lab) and loads it from local cache afterwards. A local path can be passed instead of a model name.
2. `revision` pins the exact commit of the model repository being downloaded.
3. The tokenizer converts text to tokens the model understands and back, and must match the model.
4. The `pipeline` wraps model and tokenizer for a task, here `text-generation` (other tasks include translation and summarisation). `return_full_text=False` returns only the new text rather than echoing the prompt, `max_new_tokens=500` caps response length, and `do_sample=False` makes the model pick the most likely token each time.

### 3.3 Single interaction

```python
print("What do you want?")
user_input = input()

messages = [{"role": "user", "content": user_input}]
response = generator(messages)
print(response[0]["generated_text"])
```

The input is wrapped in the chat message format (a list of role and content dictionaries) before being passed to the generator. Asked "What is the national bird of USA, and Australia?" the model answered Bald Eagle and Emu.

### 3.4 Continuous chat loop

Replacing the single interaction with a loop that runs until the user types x or X:

```python
while True:
    print("-" * 50)
    print("What do you want?")
    user_input = input("\033[92mType something, or X to exit: \033[0m")
    if user_input in ["X", "x"]:
        print("Exiting.")
        break
    messages = [{"role": "user", "content": user_input}]
    response = generator(messages)
    print(response[0]["generated_text"])
```

### 3.5 Challenge: swap the model for TinyLlama

Changing the model requires edits in three places: the model name in `AutoModelForCausalLM.from_pretrained`, the model name in `AutoTokenizer.from_pretrained`, and the `revision_id`, which is specific to each model repository.

```python
revision_id = "fe8a4ea1ffedaf415f4da2f062534de366a451e6"

model = AutoModelForCausalLM.from_pretrained(
    "TinyLlama/TinyLlama-1.1B-Chat-v1.0",
    revision=revision_id,
    device_map="auto",
    torch_dtype="auto",
    trust_remote_code=True,
)

tokenizer = AutoTokenizer.from_pretrained(
    "TinyLlama/TinyLlama-1.1B-Chat-v1.0", revision=revision_id
)
```

The rest of the program is unchanged, which shows how the pipeline abstraction makes models swappable.

## 4. Result

A working local chatbot. First run downloads the model (minutes), later runs load from cache and start quickly. Generation takes around a minute per answer on CPU. Both Phi-3-mini and TinyLlama produced sensible answers to general knowledge questions.

## 5. Lessons learned (security focus)

1. **Model supply chain is a real attack surface.** During download, transformers explicitly warned that `configuration_phi3.py` and `modeling_phi3.py` were downloaded from the model repo and should be checked for malicious code. Models on public hubs are code plus weights, not just data.
2. **`trust_remote_code=True` executes code from the model repository.** This is exactly what the warning above is about: the flag allows arbitrary Python from the hub to run on my machine. It should only be used with models from trusted publishers, and ideally after reviewing the custom code.
3. **Pinning the `revision` mitigates this.** By pinning to a specific commit hash, a later compromise of the model repository cannot silently change what I download and execute. Same principle as pinning dependency versions in `requirements.txt`: reproducibility plus supply chain protection.
4. **Generation parameters are security and reliability controls.** `max_new_tokens` caps output (cost and abuse control), and `do_sample=False` disables random sampling so the model greedily picks the most likely token, making outputs far more repeatable. LLM output is generally non deterministic, and sampling settings are one of the main levers over that.
5. **The chat message format (`role`/`content`) is the seam where prompt structure lives.** Everything the user types goes into the `user` role unchecked here; there is no input validation or system prompt. This unguarded input path is exactly what later labs on prompt injection will target.

## 6. Ideas to take forward

1. This lab is the seed for `projects/llm-chatbot/`: add a system prompt, conversation history, input handling, and basic guardrails.
2. Experiment idea: compare Phi-3-mini and TinyLlama on the same questions and note quality versus size and speed.
