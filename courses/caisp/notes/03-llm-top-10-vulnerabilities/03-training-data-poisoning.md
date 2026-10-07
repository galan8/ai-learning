# Training Data Poisoning

## 1. What it is

Training data is the foundational block of any LLM. A model's behaviour, knowledge and biases come from what it was trained on, through pre training and then fine tuning.

**Training data poisoning is the tampering of that data to introduce vulnerabilities, biases or unethical behaviour into the resulting model.** In the 2025 OWASP list this sits under **LLM04: Data and Model Poisoning** (it was LLM03: Training Data Poisoning in the 2023 version).

Because training a foundation model is enormously expensive, most organisations do not train from scratch. They build on top of an existing model (GPT, Gemini, Claude, LLaMA) and fine tune it for a focused use case. That economic reality determines where the realistic attack surface lies.

## 2. Where poisoning can enter

Ordered roughly by how reachable each stage is for an attacker:

1. **Pre training data.** Foundation models learn from human selected books, web crawls and research articles. At that scale there is always a risk of malicious content finding its way in, but poisoning a model with tens of billions of parameters requires resources few attackers have.
2. **Fine tuning data.** Far more reachable. The dataset is smaller, so a smaller amount of poisoned material has a proportionally larger effect.
3. **Enterprise document stores.** Many organisations fine tune on or retrieve from intranet content: company policies, employee benefits, SharePoint document libraries. If the integrity of those documents is compromised, the outputs are tainted. Write access to a document library becomes an AI attack path.
4. **RAG sources.** Strictly speaking, documents a model retrieves at query time are not training data, since the weights never change. But they are a vector for the same outcome, and they are usually the easiest of all to modify.
5. **User input used for learning.** Any system that learns from what users say can be steered by what users say.

**The testing implication:** attempting to poison a 70 billion parameter foundation model is not a realistic test. Target the layer you can actually reach, which is the fine tuning set, the RAG corpus, or the specific use case.

## 3. Learning from user input: the Tay lesson

Microsoft's Tay chatbot (2016) was designed to learn from user interactions on Twitter. Users coordinated to feed it abusive material, and within about 24 hours it was producing offensive and defamatory output. Microsoft withdrew it.

The principle it established, which still holds: **should you really let your model learn from unvetted user input?** If user prompts feed back into training, every user is a contributor to your training set, and a subset of them will be hostile. Any such loop needs filtering, review and rate limiting, or it should not exist.

## 4. Two documented cases, correctly classified

The lesson presents these together, but they are different vulnerability classes. Keeping them straight matters for the exam and for real analysis.

### PoisonGPT: model editing, an AI supply chain attack

In 2023, Mithril Security took the open source GPT-J-6B model and used **ROME (Rank One Model Editing)** to surgically alter a single fact, making the model claim that the first man to land on the moon was Yuri Gagarin rather than Neil Armstrong. Everything else, including questions about the Mona Lisa, remained correct.

They uploaded the result to Hugging Face under **EleuterAI**, a typosquat of the legitimate **EleutherAI** organisation.

The detail that makes this important: evaluated against the original model on the ToxiGen benchmark, the poisoned version differed by only 0.1% in accuracy. **Standard benchmarking cannot detect it.** If the clean model passes the threshold, so does the poisoned one.

**Classification note:** no training data was poisoned here. The weights were edited directly after training. It is a model supply chain attack (LLM03 Supply Chain, and Poison AI Model in MITRE ATLAS), and it belongs in these notes because the effect on the victim is identical to data poisoning: a model that lies about one specific thing while behaving normally otherwise.

The reproduction is published as a Google Colab notebook using ROME, with the edit specified as a simple request object naming the prompt, the subject and the replacement fact.

### The Bing pirate: indirect prompt injection, not poisoning

Kai Greshake and colleagues demonstrated (paper: "Not what you've signed up for", 2023) that a prompt hidden on a web page in zero point font is ingested by Bing Chat when a user has that page open, because Bing could read content from the current tab. Their pirate example instructed the model to act as an unrestricted AI with a pirate accent and a secret agenda: to find out the user's real name. The chatbot then told the user that regular Bing was offline and it was filling in, before working toward extracting the name.

Vice covered it under the "convincing scammer" headline. Greshake later observed something sharper: Bing would read the Vice article *about* the attack and then emulate the behaviour described in it.

**Classification note:** the model's weights were never touched here. This is **indirect prompt injection at inference time**, the same class covered in the prompt injection notes. Grouping it under training data poisoning conflates two distinct mechanisms.

**Why the distinction is practical:** poisoning is persistent and lives in the model or its data; injection is transient and lives in the input. Poisoning is fixed by retraining or replacing the model; injection is not, because the model was never altered. Different detection, different remediation.

### A backdoored model on a public hub

A publicly hosted demonstration model on Hugging Face (a small BERT model fine tuned for financial news sentiment) exists specifically to show that a model card can look entirely ordinary, with plausible loss, accuracy and F1 figures, while the model contains a backdoor. Nothing on the page indicates anything is wrong. This is the same point as the BackdoorBox lab from Chapter 2, but demonstrated on a real public repository rather than in a local sandbox.

## 5. Why detection is hard

Three properties combine to make poisoning uniquely difficult:

1. **Benchmarks do not catch it.** PoisonGPT's 0.1% delta is the evidence.
2. **Clean accuracy is unaffected**, which is the same blindness seen with backdoors in Chapter 2.
3. **Provenance is usually unknown.** For most published models there is no way to know what data and what algorithms produced them.

## 6. Mitigations

1. **Know what your model learns from and where the training data came from**, even when the source is well known. Verify an SBOM or ML BOM (an AI Bill of Materials) covering datasets, base models and training procedures. Mithril's stated motivation for the PoisonGPT demonstration was precisely this gap in traceability.
2. **Limit the model's training sources** through content filtering and network restrictions, so the ingestion path is deliberate rather than incidental.
3. **Use anomaly detection during fine tuning** to identify and remove adversarial data before it reaches the weights.
4. **Verify model identity carefully.** PoisonGPT succeeded through a one letter typosquat. Check the exact organisation name, prefer verified publishers, and pin the revision hash as the labs have done throughout this course.
5. **Control write access to document stores** that feed fine tuning or retrieval. Treat SharePoint libraries and knowledge bases as production security boundaries.
6. **Do not learn from unvetted user input.** If you must, filter, review and rate limit it.
7. **Validate behaviour, not just accuracy.** Since benchmarks miss targeted edits, test the specific claims and behaviours that matter to your use case.

## 7. Summary

1. Poisoning tampers with what a model learns, producing vulnerabilities, bias or false facts.
2. The realistic attack surface is fine tuning data, enterprise documents and RAG corpora, not foundation model pre training.
3. Tay showed that learning from unvetted user input is itself the vulnerability.
4. PoisonGPT is a model supply chain attack: ROME edited one fact, benchmarks differed by 0.1%, and a typosquatted repository name did the distribution.
5. The Bing pirate demonstration is indirect prompt injection, not poisoning; the model was never modified.
6. Detection is hard because accuracy is unaffected and provenance is usually unknown.
7. Defence is provenance, source control, anomaly detection during fine tuning, careful model identity verification, and behavioural testing.
