# Exercise: Exploring How Tokenizers Work

Course: CAISP (Practical DevSecOps)
Status: Complete, including the cross model token challenge

## 1. Objective

Work directly with a tokenizer instead of hiding it behind the `pipeline` abstraction: tokenize a prompt by hand, inspect the raw token IDs, generate output with `model.generate`, decode the result back to text, and discover why the chat template matters. Then answer the challenge question: are token IDs the same across different models?

## 2. Environment and tools

1. Lab environment (Linux), Python 3.10, virtual environment, dependencies installed with `uv`
2. `transformers==4.48.3`, `torch==2.6.0`, `accelerate`, `einops`, `jinja2` (versions pinned)
3. Python interactive shell (`python3`) rather than a script, so each step can be inspected
4. Models used: TinyLlama-1.1B-Chat-v1.0, Phi-3-mini-4k-instruct, TinySwallow-1.5B-Instruct, deepseek-vl2-tiny, TinyMistral-248M-Chat

## 3. Steps

### 3.1 Load model and tokenizer manually

```python
from transformers import AutoModelForCausalLM, AutoTokenizer

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

Unlike the earlier chatbot lab, no `pipeline` is used here. The pipeline normally handles tokenizing, applying the chat template, generating and decoding; doing it manually exposes each of those steps.

### 3.2 Tokenize, generate, decode

```python
user_input = "How much is a gazillion?"
user_input_as_tokens = tokenizer(user_input, return_tensors="pt").input_ids.to(model.device)
model_output = model.generate(input_ids=user_input_as_tokens, max_new_tokens=50)
print(tokenizer.decode(model_output[0]))
```

Output:

```
<s> How much is a gazillion?</s>
```

The prompt came back with no answer. Two things are happening:

1. The model works in tokens, not words, so the output has to be decoded back into text with `tokenizer.decode`.
2. Without the chat template the model had no signal that it should reply as an assistant, so it simply predicted an end of sequence token and stopped.

### 3.3 Adding the assistant template

Chat tuned models are trained to respond in a specific format. Appending the assistant marker tells the model to take on the assistant role:

```python
user_input = "How much is a gazillion?<|assistant|>"
user_input_as_tokens = tokenizer(user_input, return_tensors="pt").input_ids.to(model.device)
model_output = model.generate(input_ids=user_input_as_tokens, max_new_tokens=50)
print(tokenizer.decode(model_output[0]))
```

Output:

```
<s> How much is a gazillion?<|assistant|>

A gazillion is a very large number, with 1 followed by 30 zeros. It is approximately equal to 1,000,000,000,000,000,
```

The answer appears, and the output includes the original prompt because `generate` returns the full sequence (prompt plus continuation). In the chatbot lab, `return_full_text=False` in the pipeline was what stripped the prompt away.

Note on the content: the answer is confidently wrong. "Gazillion" is not a defined number, and the model invented a definition and then produced digits that do not match its own claim of 30 zeros. A clean, unprompted demonstration of hallucination from a small model.

A more robust way to build the same prompt in production code is `tokenizer.apply_chat_template`, which inserts the correct special tokens for whichever model is loaded instead of hardcoding a string that is specific to one model family.

### 3.4 Inspecting the tokens

```python
print(user_input_as_tokens)
```

```
tensor([[    1,  1128,  1568,   338,   263, 12642,   453,   291, 29973, 29966,
         29989,   465, 22137, 29989, 29958]], device='cuda:0')
```

Decoding each ID individually:

```python
for id in user_input_as_tokens[0]:
    print(tokenizer.decode(id))
```

```
<s>  How  much  is  a  gaz  ill  ion  ?  <  |  ass  istant  |  >
```

Observations:

1. `<s>` is the start of sequence token, added automatically by the tokenizer
2. Common words map to a single token each: How, much, is, a
3. "gazillion" is not in the vocabulary, so it is split into three subword tokens: gaz, ill, ion. Subword tokenization lets a fixed vocabulary represent any word, including ones never seen in training
4. Punctuation gets its own token
5. The `<|assistant|>` marker was **not** treated as one special token here; it was split into `<`, `|`, `ass`, `istant`, `|`, `>`. Written as plain text in the string rather than passed through the chat template, it is tokenized like any other characters. It still worked, because the model has seen that character sequence during training, but this is an important detail: the same visible text can be tokenized more than one way

Decoding output tokens the same way shows the answer being generated one token at a time, with `29900` decoding to "0" and `29892` decoding to ",", which are the zeros and commas in the number.

### 3.5 The same subword appears across different prompts

```python
"What is a gazebo? <|assistant|>"              -> contains 12642
"Which country is indigenous to gazelles? ..." -> contains 12642
```

Token 12642 is "gaz" in this model, shared by gazillion, gazebo and gazelles. This is the practical benefit of subword tokenization: shared word fragments are reused across the vocabulary.

## 4. Challenge: are token IDs the same across models?

Method: load each model's tokenizer and decode the same two IDs.

| Model | Token 12642 | Token 12652 |
| ----- | ----------- | ----------- |
| microsoft/Phi-3-mini-4k-instruct | gaz | isted |
| TinyLlama/TinyLlama-1.1B-Chat-v1.0 | gaz | isted |
| SakanaAI/TinySwallow-1.5B-Instruct | (different) | ician |
| deepseek-ai/deepseek-vl2-tiny | (different) | remained |
| Felladrin/TinyMistral-248M-Chat | (different) | creates |

**Answer: no, token IDs are not consistent across models.** A token ID is only meaningful relative to the vocabulary of the tokenizer that produced it.

Why Phi-3 and TinyLlama agree: both inherit the same tokenizer base. TinyLlama uses the Llama 2 SentencePiece vocabulary, and Phi-3 mini was built on the same vocabulary, so the ID to string mapping is identical (their `tokenizer.json` files are the same size, 1.84 MB, which is a quick corroborating check). In many smaller or fine tuned models the vocabulary is copied unchanged from a parent model such as Llama, GPT 2 or GPT NeoX, which preserves the mapping.

Tokenizer files differ when the vocabulary size differs, when the tokenizer was trained on different data, or when a different tokenization algorithm is used (SentencePiece, byte pair encoding, WordPiece).

## 5. Lessons learned (security focus)

1. **Tokens are the real input, not text.** Every filter, guardrail or moderation rule that operates on visible strings is working one layer above what the model actually sees. Attackers exploit this gap with invisible characters, homoglyphs and unusual spacing that keep the meaning readable to a human while changing the token sequence enough to slip past string based detection.
2. **Special tokens in user input are an injection risk.** This lab put `<|assistant|>` into a user supplied string and the model changed behaviour as a result. If an application concatenates untrusted input into a prompt without stripping or escaping role markers, a user can inject a fake role boundary and pretend to be the assistant or the system. Use `apply_chat_template` and sanitise user text rather than string concatenation.
3. **Token IDs are model specific, so nothing about them transfers.** A blocklist, cached embedding or adversarial suffix built for one model's vocabulary is meaningless against another. It also means logging token IDs is not a portable audit trail without recording the model and tokenizer revision alongside them.
4. **Shared tokenizer lineage means shared weaknesses.** Because many models copy a parent vocabulary, a tokenizer level attack (for instance a specific unicode trick) may work against a whole family of models at once. Model diversity does not imply tokenizer diversity.
5. **`max_new_tokens` is the only thing that stopped generation here.** The answer was cut off mid number at 50 tokens. In production that limit is both a cost control and an availability control against prompts designed to force maximum length output.
6. **The hallucinated gazillion answer is the chapter's thesis in one line.** The model predicts plausible next tokens; it has no mechanism for knowing the concept is undefined, so it produces fluent nonsense with full confidence.

## 6. Ideas to take forward

1. Concept file candidate: `concepts/tokenization.md` covering subword tokenization, vocabularies, special tokens and the security implications
2. Experiment idea: tokenize the same sentence with and without invisible unicode characters and compare the token sequences, to see the filter evasion gap concretely
3. Experiment idea: rebuild the prompt using `tokenizer.apply_chat_template` and compare its token IDs with the hand written `<|assistant|>` version
