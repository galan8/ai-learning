# What is a Large Language Model?

## Where LLMs Fit in the AI Landscape

The hierarchy: AI (divided into General AI and Narrow AI) contains Machine Learning, which in turn contains supervised learning, unsupervised learning, reinforcement learning and deep learning.

Each branch with its typical uses and techniques:

1. **Supervised learning**: used in credit scoring, email filtering and weather forecasting. Techniques: linear regression, logistic regression, decision trees, random forests.
2. **Unsupervised learning**: used in segmenting customers, detecting anomalies and noise reduction. Techniques: clustering, association, dimensionality reduction.
3. **Reinforcement learning**: used in games, robotics and drones. Techniques: value based and policy based reward learning.
4. **Deep learning**: powers natural language processing and generative AI, including text generation, image generation, audio generation and video generation. Techniques: neural networks, CNNs, RNNs and transformers.

Where LLMs fit: LLMs are part of text generation within deep learning. They generate and recognise text using pre trained transformer networks.

## Definition

An LLM is an AI system that uses deep learning techniques to understand and generate human like text. It is trained on massive amounts of text data and can process and respond to natural language inputs.

Everyday examples:

1. ChatGPT (OpenAI): proprietary
2. Claude (Anthropic): proprietary
3. Gemini (Google): proprietary
4. LLaMA (Meta): open source (more precisely open weight, since the weights are published but training data and some usage rights are restricted)
5. Amazon Bedrock: not a model itself but a managed AWS service providing access to multiple LLMs through one API

## Four Types of Language Models

1. **Statistical language models**: the earliest approach, predicting the next word from word frequency statistics in a text corpus.
2. **N-gram language models**: the classic statistical type. An n-gram is a sequence of n words, and the model predicts the next word from the previous n minus 1 words (a bigram uses one previous word, a trigram uses two). Simple and fast, but blind to anything outside its short window.
3. **Neural language models**: use neural networks and word embeddings to predict the next word, capturing meaning and longer context far better than counting statistics. RNNs and LSTMs were the main architectures of this era.
4. **Transformer language models**: the current state of the art, using the attention mechanism to consider all tokens in the input at once rather than sequentially. This is the architecture behind every modern LLM (GPT, Claude, LLaMA, Phi, Gemini).

**Causal language modeling**: strictly speaking this is a training objective rather than a separate architecture. The model predicts the next token using only the tokens to its left, never looking ahead. Transformer models trained this way are called causal language models, which is why the transformers class for loading them is named `AutoModelForCausalLM`. The contrasting objective is masked language modeling, as in BERT, where the model fills in blanked out words using context from both directions; those models are used for understanding tasks rather than text generation.

The progression from n-grams to neural to transformer models is essentially the story of how much context a model can use, and the causal objective is what makes LLMs generators rather than classifiers.
