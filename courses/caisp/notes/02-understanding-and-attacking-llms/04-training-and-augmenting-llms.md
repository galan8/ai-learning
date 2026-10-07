# Training and Augmenting LLMs

## Foundational models

LLMs are built through a two stage training process, pre training and fine tuning. The model produced by pre training is called a foundational (or base) model.

Pre training is the initial step, using a large and diverse dataset. The goal is to help the model understand language, the context of words, their relationships, and general knowledge of the world. This happens through:

1. **Pattern recognition**: recognising patterns during tokenization and embedding
2. **Contextual understanding**: understanding the context of specific words in a sentence
3. **Response generation**: recognising input patterns, understanding context, and generating a response

## Fine tuning

The model is trained further on a more specific dataset, and its weights are adjusted so it generates responses suited to a particular domain. That is the primary goal of fine tuning: making the model produce the most accurate responses within that domain. For example, a medical professional needs a high degree of precision, so fine tuning on medical material makes the model more reliable for that use.

## Retrieval Augmented Generation (RAG)

RAG keeps LLMs up to date with real world, live knowledge. It is an AI framework that merges the advantages of traditional information retrieval systems, such as search engines and databases, with the power of generative language models. By combining your own data with the world knowledge and language abilities of LLMs, RAG enables more precise, current and contextually relevant generation.

RAG is a two step process:

1. Relevant information is retrieved first
2. The answer is then generated based on that retrieval

Retrieval pulls in up to date information that was not part of the original pre trained data.

### Where retrieved information comes from, and the security question attached to each

1. **Internet access**, either through direct URL access or via a search engine. Who controls the sources? Nobody controls what content is presented, so the model can ingest attacker controlled pages. This is the route for indirect prompt injection.
2. **Direct access to databases and internal context**, including relational databases and vector databases. Who controls the access? These stores can contain PII, financial, medical and other private information, so retrieval permissions become an access control problem.

## Key distinction

Fine tuning changes the model's weights; RAG changes what the model sees at inference time. Fine tuning teaches behaviour and domain style; RAG supplies current facts. They solve different problems and have different attack surfaces.
