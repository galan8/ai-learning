# How LLMs Work

## High Level

**Training**: LLMs process massive volumes of text from the internet, books and other sources. Models contain billions to trillions of parameters and consume enormous amounts of compute to train.

**Architecture**: LLMs are based on the transformer neural network architecture, which uses attention based mechanisms to weigh how much each part of the input relates to every other part.

**Processing**: text is processed as sequences of tokens (words or word fragments). During training the model learns patterns and relationships in language from these sequences.

**The key insight**: even though an LLM may look like it possesses actual understanding, in reality it is extremely good at predicting the next syntactically and statistically likely token given everything that came before. Fluency is not comprehension.

**Security relevance**: that last point is the root of much of this chapter. Because the model only predicts plausible continuations, it has no built in notion of truth (hallucinations), no inherent separation between instructions and data (prompt injection), and it will fluently continue harmful text if steered there (jailbreaks). Everything the model "knows" is statistical pattern, which attackers can exploit.

## Low Level: the Four Phases

An LLM's life has four phases, split across two stages:

1. Tokenizing, embedding and transforming happen during **training**
2. Generating happens during **inference** (interacting with the model)

Tokenizing and transforming also occur at inference time: every prompt is tokenized before the model can process it, and the transformer runs on it to produce output. The cleanest way to hold it: all four phases exist, training is where the model learns (embedding weights are set, transformer weights are adjusted), and inference is where the learned model is used.

The pipeline, phase by phase:

1. **Tokenizing**: the input text is broken into tokens and every token is converted to a number (its token ID). Tokens are often whole words but can be word fragments or punctuation. This is how language becomes something a neural network can compute on.
2. **Embedding**: each token is mapped to a vector that captures its meaning based on the training data, so that similar words end up closer together in the embedding space. Visualised as a two dimensional graph, "king" and "queen" would sit near each other while "bicycle" sits far away. These associations are learned during training and stored as weights in the model. (Related but distinct: a vector database stores embeddings outside the model, which is how RAG retrieval works; inside the LLM itself the embeddings live in the model's weight matrices.)
3. **Transforming**: the transformer layers process the embedded sequence using attention, weighing how every token relates to every other token, to predict the next token. During training this prediction is compared to the real next word and the weights are corrected; during inference the prediction becomes the output.
4. **Generating**: the model produces the response one token at a time, feeding each predicted token back in to predict the next, until an entire sentence or answer is generated.

**Security perspective**: the essential split is training versus inference, with different algorithms involved across the four phases. Each stage is a different attack surface: training time attacks target the data and learned weights (data poisoning, backdoors), while inference time attacks target the prompt and generation process (prompt injection, jailbreaks, model extraction). Defences differ accordingly, which is why the split is worth keeping in view.

## Connection to the labs

Phi-3-mini and TinyLlama are small open weight LLMs; the tokenizer loaded in the chatbot and tokenizer labs is the component that turns text into the token sequences described here; and `max_new_tokens` caps exactly this next token prediction loop.
