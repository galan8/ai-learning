# Retrieval Augmented Generation (RAG)

RAG is an AI framework that merges the advantages of traditional information retrieval systems (such as search engines and databases) with the power of large language models. By combining your own data with the world knowledge and language abilities of LLMs, RAG enables more precise, current and contextually relevant generation tailored to your specific needs.

## The two components

1. **Retrieval models**: perform information retrieval or semantic search to find the most pertinent data for a query, typically from your own documents or databases
2. **Generative models (LLMs)**: produce new, coherent text based on a prompt. On their own they struggle with factual accuracy and staying current, which leads to AI hallucinations.

## How RAG works

When a query arrives, relevant documents are retrieved first and then passed to the LLM along with the prompt, so the model generates its answer grounded in that retrieved content.

## Benefits

1. **Access to fresh information**: overcomes the LLM's reliance on outdated training data with a fixed cutoff
2. **Factual grounding**: outputs are based on actual, relevant facts, reducing hallucinations and improving factual accuracy
3. **Efficient search**: uses vector databases and relevance re rankers to retrieve semantically similar documents, often combined with hybrid search methods (keyword plus semantic)
4. **Tailored responses**: answers reflect your organisation's own data and use cases rather than generic knowledge

RAG reduces inconsistencies by grounding LLM outputs in accurate, retrieved information, and it underpins context aware agents and chatbots tailored to specific data and domains.

**Security relevance:** RAG introduces its own attack surface, including poisoning of the knowledge base (indirect prompt injection through retrieved documents), leakage of sensitive data stored in vector databases, and manipulation of retrieval results to steer model outputs.
