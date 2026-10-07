# Exercise: Building a Summarizer Tool Using an LLM

Course: CAISP (Practical DevSecOps)
Status: Complete (all six steps, including file and URL input)

## 1. Objective

Build a text summarisation tool using a sequence to sequence model rather than a causal one, then extend it to accept content from local files and remote URLs instead of typed input. The lab also demonstrates what happens when input exceeds the model's limits.

## 2. Environment and tools

1. Lab environment (Linux), Python virtual environment, dependencies installed with `uv`
2. `transformers`, `torch`, `accelerate`, `einops`, `jinja2` (pinned), plus `requests` for URL fetching
3. Model: `Falconsai/text_summarization`, a small model fine tuned specifically for summarisation
4. GPU available in this lab (`Device set to use cuda:0`), unlike the CPU only chatbot lab

## 3. Steps

### 3.1 A different model class

```python
from transformers import AutoModelForSeq2SeqLM, AutoTokenizer, pipeline

revision_id = "6e505f907968c4a9360773ff57885cdc6dca4bfd"
model = AutoModelForSeq2SeqLM.from_pretrained(
    "Falconsai/text_summarization",
    revision=revision_id,
    device_map="auto",
    torch_dtype="auto",
    trust_remote_code=True,
)
tokenizer = AutoTokenizer.from_pretrained("Falconsai/text_summarization", revision=revision_id)

summarizer = pipeline(
    "summarization",
    model=model,
    tokenizer=tokenizer,
    max_length=150,
    min_length=30,
    do_sample=False,
)
```

Key difference from the chatbot lab: `AutoModelForSeq2SeqLM` instead of `AutoModelForCausalLM`.

1. **Causal LM** (Phi 3, TinyLlama): decoder only, predicts the next token from what came before, built for open ended generation
2. **Seq2Seq LM** (this model, based on T5): encoder plus decoder, reads a complete input then produces a transformed output, built for tasks with a defined input and output such as summarisation and translation

How to know which class a model needs: the model card on Hugging Face has a "Use this model" button that shows the correct loading code.

The pipeline task is `"summarization"` rather than `"text-generation"`, and its parameters change accordingly: `max_length` and `min_length` bound the summary, and `do_sample=False` keeps output deterministic.

### 3.2 Basic summarizer

```python
user_input = input("Enter the text you want to summarize:")
response = summarizer(user_input)
print(response[0]["summary_text"])
```

Note the output key is `summary_text`, not `generated_text` as in the text generation pipeline.

Given a paragraph about cormorants, the model returned a condensed version keeping the size, diet and diving details. This is **extractive style** output: the summary is largely composed of sentences lifted from the source rather than reworded, which is typical of a small T5 based summariser.

### 3.3 Continuous loop

```python
while True:
    print("-" * 50)
    user_input = input("\033[92mEnter the text you want to summarize: \033[0m")
    if user_input in ["X", "x"]:
        print("Exiting.")
        break
    print("-" * 50)
    print("Calling LLM for summarizing")
    response = summarizer(user_input)
    print(response[0]["summary_text"])
```

Same pattern as the chatbot lab: loop until the user types x or X.

### 3.4 Reading from files and URLs

Two input sources, selected by prefix:

```python
import requests

while True:
    print("-" * 50)
    user_input = input("\033[92mEnter a file path (file://) or a URL (http:// or https://): \033[0m")

    if user_input in ["X", "x"]:
        print("Exiting.")
        break

    if user_input.startswith("file://"):
        with open(user_input[7:], "r") as file:      # strip the file:// prefix
            user_input = file.read()
    elif user_input.startswith("http://") or user_input.startswith("https://"):
        response = requests.get(user_input)
        user_input = response.text
    else:
        print("Invalid input. Please start with file:// or http://https://.")
        continue

    print("-" * 50)
    print("Calling LLM")
    response = summarizer(user_input)
    print(response[0]["summary_text"])
```

Tested against downloaded sample files (Saturnalia.txt, Mephistopheles.txt, VincentVanGogh.txt) and raw text URLs (Cormorants, Albatross, Conures, BallPython, Snakes).

## 4. Results and failures

1. Small and medium inputs summarised correctly (Saturnalia, Mephistopheles, Cormorants, Albatross)
2. Mephistopheles triggered a warning: the token sequence (1115 tokens) exceeded the model's maximum sequence length of 512. The pipeline truncated the input, so the summary reflected only the first portion of the document, silently
3. VincentVanGogh.txt and the larger URL contents (BallPython, Snakes) crashed the program with `torch.OutOfMemoryError: CUDA out of memory`, because there is no exception handling and no chunking

The 512 token limit is the important constraint here: this model can only see roughly 400 words at a time. Real summarisers handle longer documents by chunking the text, summarising each chunk, then summarising the summaries (map reduce), or by using a model with a longer context window.

## 5. Lessons learned (security focus)

This lab is the most security relevant so far, because it is the first one where the model consumes content the user did not type.

1. **This is the indirect prompt injection pattern in miniature.** The moment a program fetches a file or web page and feeds it to a model, the content author becomes an input source. Nothing in this code distinguishes instructions from data, so a hosted page containing text aimed at the model is processed exactly like ordinary prose. This is precisely the MITRE ATLAS technique behind the plugin compromise example in the Chapter 2 notes.
2. **`file://` with no path validation is a local file read primitive.** The code strips seven characters and passes the rest straight to `open()`. `file://../../etc/passwd` or any absolute path works. In a terminal tool that is your own file; wrapped in a web service, it is arbitrary file disclosure.
3. **`requests.get()` on user supplied URLs is textbook SSRF.** There is no allowlist, no scheme restriction beyond the prefix check, and no block on internal addresses, so the tool will happily fetch `http://169.254.169.254/` (cloud instance metadata) or internal services and print the result through the model. Combined with point 2, the input handling here is a compact catalogue of what not to do.
4. **Silent truncation is an integrity problem.** The 512 token limit produced a warning, not an error, and the program carried on to summarise only the opening section. A user reading the summary has no indication that most of the document was ignored. Anyone relying on such a summary for a decision (contract review, incident triage) is being misled by omission.
5. **Unbounded input is a denial of service vector.** A large document crashed the process outright with an out of memory error. In a service, an attacker who controls the input size can crash workers or exhaust GPU memory on demand. Input size limits belong in front of the model, not in a try block after it.
6. **Missing exception handling is a security issue, not just an ergonomics one.** The crash printed a full CUDA memory dump including device capacity and allocation details, which is information disclosure about the deployment.

## 6. Ideas to take forward

1. Experiment idea: harden this tool. Add path validation to reject traversal and restrict to a working directory, add a URL allowlist plus a block on private and link local ranges, cap the download size, and wrap generation in exception handling
2. Experiment idea: implement chunking so long documents summarise fully rather than silently truncating, and compare against the truncated output
3. Concept file candidate: `concepts/causal-vs-seq2seq-models.md` on when to use each architecture
4. Concept file candidate: `concepts/indirect-prompt-injection.md`, using this lab as the concrete example of the untrusted content path
