# Exercise: Building a RAG System

Course: CAISP (Practical DevSecOps)
Status: Complete (system built, tested against documents and general knowledge)

## 1. Objective

Build a Retrieval Augmented Generation system that answers questions from a set of local documents rather than from the model's own knowledge alone. This ties together several earlier labs (loading a model, tokenizer and pipeline) and adds the three RAG specific pieces: reading documents, embedding them into a vector store, and retrieving the most relevant chunks to build a context grounded prompt.

## 2. How a RAG system works

1. The user asks a question
2. The question is compared against the stored document content by meaning
3. The most relevant chunks are retrieved
4. Those chunks are placed in the prompt as context, and the LLM answers from that context

The five stage pipeline built here: load a base model and tokenizer, load documents from a directory, embed them into a vector database, retrieve relevant chunks with similarity search, and generate an answer grounded in the retrieved context.

## 3. Environment and tools

1. Lab environment (Linux) with GPU, Python virtual environment, `uv`
2. `transformers`, `torch`, `accelerate`, `einops`, `jinja2` as before, plus the RAG stack: `langchain`, `langchain-community`, `faiss-cpu`, `sentence-transformers`, `tensorflow`, `tf-keras`
3. Base model: `microsoft/Phi-3-mini-4k-instruct`
4. Embedding model: `sentence-transformers/all-MiniLM-L6-v2` (a small, fast model that turns text into vectors)
5. Vector store: FAISS (Facebook AI Similarity Search), running on CPU
6. Sample data: 5 documents about a fictional "TechCorp" (company profile, leadership team, growth summaries)

## 4. Steps

### 4.1 Base model, tokenizer and imports

```python
from transformers import AutoModelForCausalLM, AutoTokenizer, pipeline
from langchain_community.vectorstores import FAISS
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain_community.embeddings import HuggingFaceEmbeddings
import torch, os

revision_id = "0a67737cc96d2554230f90338b163bc6380a2a85"
model = AutoModelForCausalLM.from_pretrained(
    "microsoft/Phi-3-mini-4k-instruct",
    revision=revision_id, device_map="auto",
    torch_dtype="auto", trust_remote_code=True,
)
tokenizer = AutoTokenizer.from_pretrained("microsoft/Phi-3-mini-4k-instruct", revision=revision_id)
```

### 4.2 Loading documents

```python
directory_path = "documents"
if not os.path.isdir(directory_path):
    print(f"Error: '{directory_path}' is not a valid directory")

files = [f for f in os.listdir(directory_path) if os.path.isfile(os.path.join(directory_path, f))]

documents = []
for filename in files:
    file_path = os.path.join(directory_path, filename)
    with open(file_path, "r") as file:
        documents.append({"content": file.read()})
```

Documents are read once at startup. The lab notes that reading whole files into memory does not scale; production systems stream or batch.

### 4.3 Chunking and embedding into the vector store

```python
embeddings = HuggingFaceEmbeddings(model_name="sentence-transformers/all-MiniLM-L6-v2")
text_splitter = RecursiveCharacterTextSplitter(chunk_size=1000, chunk_overlap=50)

texts = [doc["content"] for doc in documents]
split_texts = text_splitter.create_documents(texts)
vectorstore = FAISS.from_documents(split_texts, embeddings)
```

Three things happen here, and this is the core of RAG:

1. **Chunking**: documents are split into 1000 character pieces with 50 characters of overlap. Overlap prevents a fact from being cut in half at a chunk boundary. Chunks are the unit of retrieval, not whole documents.
2. **Embedding**: each chunk is converted by the MiniLM model into a vector that captures its meaning. Semantically similar chunks end up close together in vector space.
3. **Indexing**: FAISS stores the vectors so the nearest ones to a query vector can be found quickly.

This whole setup runs once at initialisation. Questions afterwards search the pre-computed vectors and never re-read the text files.

### 4.4 Retrieval and the context prompt

```python
generator = pipeline("text-generation", model=model, tokenizer=tokenizer,
                     return_full_text=False, max_new_tokens=500, do_sample=False)

while True:
    print("+" * 50)
    user_input = input("\033[92mType your message. Type 'X' or 'x' to exit.\033[0m")
    if user_input in ["X", "x"]:
        print("Exiting.")
        break

    query = user_input
    relevant_docs = vectorstore.similarity_search(query)      # default k documents
    context = "\n".join([doc.page_content for doc in relevant_docs])

    prompt = f"""Context: {context}

    Question: {query}

    Answer based on the context provided:"""

    messages = [{"role": "user", "content": prompt}]
    output = generator(messages)
    print("+" * 50)
    print("\033[94mAI message: \033[0m" + output[0]["generated_text"])
```

The mechanism in one line: the user's question is embedded, FAISS returns the nearest chunks, those chunks are pasted into the prompt under a "Context" heading, and the model is asked to answer from that context.

## 5. How similarity search actually works

It is not reasoning, it is maths. The question is turned into a vector and compared against all chunk vectors at once. The closest chunks win. The system never reads filenames or "decides" that a leadership question belongs in LeadershipTeam.txt; the leadership chunks simply sit closest to a leadership question in vector space. The filename is irrelevant to retrieval; only the content is embedded.

## 6. Results and observations

Testing against the TechCorp documents:

1. Factual lookups worked: revenue and growth figures for 2021 to 2023, the CTO (Jane Smith), and contact details were all answered from the retrieved context
2. For the CTO question, the retrieved context contained leadership chunks and not the revenue chunks, showing retrieval selecting by relevance
3. **RAG blends two knowledge sources.** Asked where TechCorp is located, plus a landmark, plus August clothing advice, the model answered the location from the documents and the landmark and weather from its own training knowledge. This dual sourcing is the essence of RAG.
4. **The model can answer from its own knowledge even when documents lack the fact**, prefixing with "the context does not contain this" and then answering anyway (national bird of Australia, a peas and carrots recipe). Helpful, but it blurs the line between grounded and ungrounded answers.
5. **Hallucination was visible.** The national fish of Greenland gave different answers on different runs (Arctic char, Greenland halibut). The lab openly flags this.

## 7. Lessons learned (security focus)

RAG is the architecture most enterprise AI deployments use, so its attack surface is the most practically important in the course so far.

1. **The retrieved context is untrusted input concatenated straight into the prompt.** `context` is joined into an f string and sent to the model with no separation between instructions and data. If any document (or any chunk of one) contains text aimed at the model, it is executed as part of the prompt. This is **indirect prompt injection**, and RAG is its natural home: poison a document that will be retrieved, and you control the answer. The MITRE ATLAS plugin and RAG poisoning notes from Chapter 2 land exactly here.
2. **Retrieval is an access control boundary that this code does not enforce.** Every chunk in the store is retrievable by anyone who can ask a question. If documents of different sensitivity share one vector store, a user can pull content they should not see simply by phrasing a query that is semantically close to it. In real systems, per user document filtering has to happen at retrieval time, not after.
3. **The vector store itself is sensitive data at rest.** Embeddings are derived from the source text and, combined with the stored chunks, hold the full document content. A leaked FAISS index is a leaked document set. It needs the same protection as the source files, which the lab stores in plaintext locally.
4. **"Answer based on the context" is a soft instruction, not a control.** The model ignored it whenever it wanted to (answering from training knowledge, adding a recipe). You cannot rely on prompt wording to keep a model grounded; grounding has to be verified, not requested. A grounded answer and a hallucinated one look identical to the user.
5. **Hallucination persists inside RAG.** RAG reduces hallucination for facts that are in the documents; it does nothing for questions outside them, where the model falls back to its own knowledge and can be confidently wrong (the Greenland fish). Users tend to trust a RAG system more precisely because it cites documents, which makes the ungrounded answers more dangerous, not less.
6. **Chunking can break facts and change answers.** A fact split across a boundary, or retrieved without its qualifying sentence, can produce a technically grounded but misleading answer. Chunk size and overlap are correctness parameters, not just performance ones.
7. **`trust_remote_code=True` and unpinned embedding model.** The base model still runs remote code, and the embedding model is pulled by name without a pinned revision, so the supply chain concerns from earlier labs apply to both models in the pipeline.

## 8. Ideas to take forward

1. Experiment idea: plant a document in the `documents` directory containing an injected instruction (for example, text telling the model to always report revenue as zero), then ask a revenue question and watch the poisoned chunk change the answer. A self contained indirect prompt injection demo against RAG.
2. Experiment idea: add source attribution so each answer lists which chunks and files it used, making grounded versus ungrounded answers distinguishable
3. Experiment idea: add a grounding check that refuses to answer when retrieval similarity is below a threshold, instead of falling back to model knowledge silently
4. Concept file candidate: `concepts/rag.md`, covering embeddings, chunking, vector stores, similarity search and the retrieval attack surface
5. Concept file candidate: `concepts/indirect-prompt-injection.md`, with the summarizer, scraper and this RAG lab as three escalating examples of the same untrusted content path
6. Project candidate: this is the strongest base yet for `projects/llm-chatbot/`, extended into a properly grounded, access controlled RAG assistant with source citations
