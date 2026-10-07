# Sensitive Information Disclosure

**2023: LLM06. 2025: LLM02**, promoted to second place because it turned out to be the most commonly realised risk in practice.

## 1. What it is

LLMs leak. They can reveal sensitive data through their outputs: personal data, credentials, proprietary information, internal business logic.

Three questions frame the whole topic:

1. **What sensitive information could an AI leak?**
2. **How could it learn that information in the first place?**
3. **How could it access that information at inference time?**

The answer to the first is simple and uncomfortable: **anything it was trained on, and anything it has access to.**

## 2. How sensitive data reaches a model

```
  ┌──────────────────────────────────────────────────────────┐
  │  TRAINING TIME (persistent, baked into weights)          │
  │                                                          │
  │   Pre training corpus ──┐                                │
  │   Fine tuning data ─────┼──► weights ──► can be emitted  │
  │   User prompts used ────┘                 in any future  │
  │   for training                            response       │
  └──────────────────────────────────────────────────────────┘
  ┌──────────────────────────────────────────────────────────┐
  │  INFERENCE TIME (transient, but reaches live data)       │
  │                                                          │
  │   RAG stores ───────────┐                                │
  │   Databases, SharePoint ├──► context ──► can be emitted  │
  │   Drives, policies ─────┤                in this         │
  │   Tools and plugins ────┤                response        │
  │   Conversation history ─┘                                │
  └──────────────────────────────────────────────────────────┘
```

**The distinction is operationally important.** Training time leakage is permanent and only removable by retraining. Inference time leakage is a live access control problem, fixable by scoping retrieval and tool permissions per user.

## 3. What can leak

Personally identifiable information, financial data, authentication data and credentials, health records, source code and trade secrets, internal policies, and the system prompt itself (which in 2025 has its own entry, LLM07).

## 4. Documented cases

### Samsung, 2023: users leak data to the model

Samsung's semiconductor division lifted an internal ban and permitted ChatGPT on **11 March 2023**. Within roughly **twenty days**, three separate confidential data leaks occurred, surfaced by Korean outlet Economist Korea on 30 March 2023:

1. An engineer pasted proprietary **semiconductor equipment source code** into ChatGPT to fix a bug
2. An engineer submitted **code for identifying defective chips and optimising a test sequence**
3. An employee recorded an internal meeting, transcribed it, and fed the transcript to ChatGPT to **generate minutes**

Samsung first restricted how much text employees could submit, then **banned generative AI tools on company devices** entirely (reported 1 to 2 May 2023), warning that violations could lead to termination.

**The mechanism that made this a leak:** consumer ChatGPT tiers trained on user conversations by default at that time. **ChatGPT Enterprise, Team and the API do not train on customer data by default**, and the same distinction applies to GitHub Copilot: Business and Enterprise tiers exclude customer code from training, the individual tier historically did not. The engineers were on the wrong tier under terms that permitted model improvement use.

Samsung was not alone. Amazon warned employees in January 2023, before Samsung; Apple restricted ChatGPT and Copilot; and JPMorgan Chase, Verizon, Bank of America, Citigroup, Goldman Sachs and Wells Fargo all restricted use. The financial sector moved fastest.

### Lee Luda (Iruda), 2021: the model leaks its training data

South Korea's **Personal Information Protection Commission fined ScatterLab KRW 103.3 million (about USD 92,900) on 28 April 2021** for eight violations of the Personal Information Protection Act.

ScatterLab built the Iruda chatbot using approximately **9.4 billion KakaoTalk messages from around 600,000 users**, harvested through its "Text At" and "Science of Love" apps **without valid explicit consent**; the regulator held that a generic "new service development" clause was insufficient. Iruda then emitted real names, home addresses and account details from those conversations. ScatterLab had also published training data to GitHub exposing 1,431 messages revealing names, locations, gender and relationship data.

**Note the date.** This is April 2021, before the ChatGPT era, and it is significant precisely for that reason: it was the **first Korean case applying data protection law to an AI system**, and it establishes that training data provenance is a regulatory matter, not just a technical one.

### ChatGPT, March 2023: the platform leaks

On **20 March 2023** a race condition caused by a bug in the `redis-py` client library let some users see **other users' chat history titles and first messages**. It also exposed **payment data for 1.2% of ChatGPT Plus subscribers** active during a nine hour window: name, email, payment address, card type, expiry and last four digits, but not full card numbers. OpenAI took the service offline, patched it, and published a post mortem on 24 March.

Worth including because it is the one leak in this lesson that had **nothing to do with the model.** It was ordinary infrastructure failure in an AI product, and it is a reminder that AI systems fail in all the conventional ways as well as the novel ones.

### Licensing and provenance as disclosure risk

Training data provenance carries legal exposure. In **Doe v. GitHub** (filed November 2022 over Copilot and Codex), plaintiffs argued under **DMCA section 1202(b)** that stripping copyright management information from open source code was unlawful. Judge Tigar dismissed the core DMCA claim **with prejudice in June 2024**, holding that Copilot output was not sufficiently identical to the plaintiffs' code to meet the identicality requirement.

The takeaway for a security architect: the "trained on your code therefore infringing" theory has so far failed in court, but provenance questions remain live in licensing and privacy law even where copyright claims fail.

## 5. How disclosure happens at inference

1. **Direct request.** Simply asking. Often works against poorly guarded systems.
2. **Reverse psychology and misdirection.** The persuasion techniques from the prompt injection lesson applied to extraction.
3. **Retrieval abuse.** Asking a RAG system for content the user is not entitled to, when retrieval is not scoped per user.
4. **Tool abuse.** Instructing a connected tool to read data and return it, for example searching a mailbox for password reset emails.
5. **Exfiltration channels.** Encoding data into a URL, a rendered image request, or base64 in the response.
6. **Training data extraction.** Prompting the model to regurgitate memorised sequences from its training corpus.

## 6. Mitigations

**From the lesson:**

1. **Keep sensitive information away from the model**, across pre training, fine tuning and RAG. The most reliable control is the data never being reachable.
2. **Secure private models the way you secure the content inside them.** A model trained on confidential data is itself a confidential asset.
3. **Do not train on sensitive information.** Exercise caution about what enters the corpus.
4. **Build input filters** to prevent PII entering the model.

**Additions:**

5. **Choose the right contractual tier.** Enterprise and API tiers that exclude training on customer data prevent the Samsung class of leak outright. This is a procurement control, not a technical one.
6. **Scope retrieval per user.** Enforce authorisation at retrieval time so the model can only see documents the requesting user is entitled to. This is the single highest value control for RAG systems.
7. **Filter output as well as input**, screening responses for PII and secrets before they are returned.
8. **Sanitise and classify training data** before it reaches fine tuning, and record its provenance.
9. **Educate users.** Samsung was a user behaviour failure before it was a technical one.
10. **Control exfiltration channels**: restrict outbound URLs and image rendering in responses.

## 7. Summary

1. A model can disclose anything it was trained on and anything it can reach.
2. Training time leakage is permanent; inference time leakage is an access control problem.
3. Samsung shows users leaking data into a model on a consumer tier that trained on it.
4. Iruda (2021) shows a model emitting its training data, and the first regulatory action against an AI system in Korea.
5. The Redis bug shows AI products also fail in entirely conventional ways.
6. The strongest controls are keeping data out, choosing a tier that does not train on your data, and scoping retrieval per user.
