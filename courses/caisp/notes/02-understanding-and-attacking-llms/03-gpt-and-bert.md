# GPT and BERT

## GPT (Generative Pre trained Transformer)

GPT models are built on the transformer neural network architecture.

They perform tasks such as answering questions, summarising text, generating content and completing sentences, delivering answers in a conversational style and enhancing search. Image and music generation come from separate generative models; recent GPT versions are multimodal and can accept or produce images, but that is a property of those specific models, not of text generation itself.

The transformer architecture was introduced in 2017 by a team of researchers at Google in the paper "Attention Is All You Need". The transformer uses self attention mechanisms to process and generate text, which lets it handle long range dependencies across a sequence.

Two stage build:

1. **Pre training**: the model is trained on a large corpus of text from diverse sources
2. **Fine tuning**: the pre trained model is then trained further on a more specific dataset

Generative capabilities: generating text such as articles and code snippets, answering questions, and completing sentences.

### Evolution of the GPT family

1. **GPT 1**: introduced the pre training plus fine tuning approach
2. **GPT 2**: 1.5 billion parameters, known for markedly improved text generation
3. **GPT 3**: 175 billion parameters, a significant advance in natural language ability
4. **GPT 4 and later**: better performance, understanding and contextual coherence; parameter counts are no longer published by OpenAI

More parameters generally means more capacity to represent patterns from the training data, and therefore broader knowledge and better handling of complex prompts. Size alone is not the whole story: data quality, training method and fine tuning matter as much, and modern smaller models often outperform older larger ones.

### Neural network architecture

Flow through the network:

1. **Input layer**: raw data enters the network
2. **Hidden layers**: successive transformations are applied, and each connection carries an associated weight representing learned relationships such as the proximity and association of words
3. **Output layer**: produces the prediction or decision

Model weights are crucial for LLMs, because the weights are what let the model predict the most probable next word. The weights are the learned knowledge of the model; everything the model "knows" lives there.

Architectural components found in these networks: feed forward layers (the standard processing layers inside each transformer block), convolutional networks (used mainly in vision models rather than LLMs), and residual connections (skip connections that let signals bypass layers, which makes very deep networks trainable).

**The most important property**: GPT models are forward only. They predict the next token based on what has already been generated, as in "The cat sat on the ...". This is causal language modeling in practice.

Applications: conversational agents, content creation, programming assistance, education and training.

## BERT (Bidirectional Encoder Representations from Transformers)

BERT is a model developed by Google that understands the context of words in a sentence by looking at what comes both before and after them. Unlike previous models that read text in a single direction, BERT processes text bidirectionally, allowing a deeper understanding of meaning and context.

Key features:

1. **Bidirectional understanding**: it sees the words to the left and to the right of each position
2. **Pre training on large datasets** using two objectives: masked language modeling (predicting blanked out words) and next sentence prediction
3. **Fine tuning** on specific datasets for particular tasks

**Key contrast**: GPT predicts the next word and is therefore built for generation; BERT is not meant for text generation, it is built for understanding. This is the encoder versus decoder distinction: BERT is encoder only (understanding), GPT is decoder only (generation).

Applications of BERT: text classification tasks (spam detection, sentiment analysis), question answering, named entity recognition, and improving search and information retrieval (Google uses BERT in search).
