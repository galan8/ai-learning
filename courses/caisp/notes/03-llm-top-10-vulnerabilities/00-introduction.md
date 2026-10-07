# Introduction to the OWASP Top 10 LLM Attacks

## What the list is

The **OWASP Top 10 for Large Language Model Applications** is maintained by the **OWASP GenAI Security Project**. It catalogues the most critical vulnerability classes in LLM based systems, each with description, example attack scenarios and mitigations.

It is the common vocabulary for this field. When an auditor, a vendor questionnaire or a job interview refers to "LLM01", this is what they mean.

## The version problem, and why it matters

This chapter teaches the **2023 (v1.1)** list. The current standard is the **2025 (v2.0)** list, published **18 November 2024**. Three of the ten topics in this chapter **no longer exist as standalone entries**.

Keep both in view: the concepts remain valid threats, but the numbering you quote must match the version you are working against.

### The 2023 list (what these lessons follow)

| ID | Name |
| -- | ---- |
| LLM01 | Prompt Injection |
| LLM02 | Insecure Output Handling |
| LLM03 | Training Data Poisoning |
| LLM04 | Model Denial of Service |
| LLM05 | Supply Chain Vulnerabilities |
| LLM06 | Sensitive Information Disclosure |
| LLM07 | Insecure Plugin Design |
| LLM08 | Excessive Agency |
| LLM09 | Overreliance |
| LLM10 | Model Theft |

### The 2025 list (current)

| ID | Name |
| -- | ---- |
| LLM01 | Prompt Injection |
| LLM02 | Sensitive Information Disclosure |
| LLM03 | Supply Chain |
| LLM04 | Data and Model Poisoning |
| LLM05 | Improper Output Handling |
| LLM06 | Excessive Agency |
| LLM07 | System Prompt Leakage |
| LLM08 | Vector and Embedding Weaknesses |
| LLM09 | Misinformation |
| LLM10 | Unbounded Consumption |

### Mapping 2023 to 2025

| 2023 | 2025 | What changed |
| ---- | ---- | ------------ |
| LLM01 Prompt Injection | LLM01 Prompt Injection | Unchanged, still number one |
| LLM02 Insecure Output Handling | **LLM05** Improper Output Handling | Renamed and moved down |
| LLM03 Training Data Poisoning | **LLM04** Data and Model Poisoning | Broadened to cover model poisoning, not just data |
| LLM04 Model Denial of Service | **LLM10** Unbounded Consumption | Broadened from availability to include cost (denial of wallet) |
| LLM05 Supply Chain Vulnerabilities | **LLM03** Supply Chain | Promoted |
| LLM06 Sensitive Information Disclosure | **LLM02** Sensitive Information Disclosure | Promoted to second, reflecting real world prevalence |
| LLM07 Insecure Plugin Design | **removed** | Distributed across Excessive Agency (LLM06) and Supply Chain (LLM03) |
| LLM08 Excessive Agency | **LLM06** Excessive Agency | Promoted |
| LLM09 Overreliance | **LLM09** Misinformation | Renamed and refocused onto the model generating falsehood, with overreliance as a contributing factor |
| LLM10 Model Theft | **removed** | Distributed across Supply Chain (LLM03) and Excessive Agency (LLM06) |
| | **LLM07** System Prompt Leakage | **New** |
| | **LLM08** Vector and Embedding Weaknesses | **New**, reflecting the shift to RAG |

### What the changes tell you

The two additions and three promotions describe an architectural shift. In 2023 the typical deployment was a chat interface over a model. By 2025 it is a **RAG pipeline driven by tool calling agents**, so OWASP added an entry for vector stores, added one for system prompt leakage, and promoted the entries about agency, supply chain and data disclosure.

Weight the list to your architecture:

1. **Chat application:** LLM01 prompt injection, LLM02 sensitive information disclosure, LLM09 misinformation
2. **RAG system:** add LLM08 vector and embedding weaknesses
3. **Agentic system:** weight LLM06 excessive agency, LLM03 supply chain, LLM10 unbounded consumption

## How the ten relate to each other

They are not ten independent problems. They form chains:

```
   ENTRY                    PROPAGATION                 IMPACT
   ─────                    ───────────                 ──────

   Prompt Injection ──┐
   (direct or         │
    indirect)         ├──► Excessive Agency ──────► actions taken
                      │    (what the model            in the world
   Poisoned data ─────┤     is permitted to do)
   or model           │
                      ├──► Improper Output ────────► XSS, RCE, SQLi,
   Supply chain ──────┘    Handling (what             SSRF in the
   compromise              consumes the output)       consuming system

                           Sensitive Information ───► data loss
                           Disclosure

                           Misinformation ──────────► wrong decisions,
                                                      legal liability
```

Two principles follow, and they run through every lesson in this chapter:

**Injection is the way in; output handling and agency decide the blast radius.** An injection that reaches nothing does nothing.

**Impact is a design choice.** You cannot fully prevent prompt injection, because an LLM has no parser separating instructions from data. You can decide what a successful injection is able to reach.
