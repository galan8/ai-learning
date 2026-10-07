# Chapter 3 Summary

## The argument in one line

An LLM cannot separate instructions from data, cannot distinguish truth from plausibility, and executes with whatever privileges you grant it; the ten entries in this list are the consequences of those three facts meeting real architectures.

## The ten, and what each actually is

| # (2023) | Entry | The one sentence version |
| -------- | ----- | ------------------------ |
| LLM01 | Prompt Injection | Data treated as command, with no parser to separate them |
| LLM02 | Insecure Output Handling | Model output is untrusted input to whatever consumes it |
| LLM03 | Training Data Poisoning | The model learns whatever correlations the data contains, including planted ones |
| LLM04 | Model Denial of Service | A cheap prompt forcing an expensive response |
| LLM05 | Supply Chain | You inherit the security of everything upstream of you |
| LLM06 | Sensitive Information Disclosure | The model can emit anything it was trained on or can reach |
| LLM07 | Insecure Plugin Design | A plugin is an API whose caller is a persuadable model |
| LLM08 | Excessive Agency | Least privilege failure applied to a probabilistic component |
| LLM09 | Overreliance | Fluent output carries no signal of its own accuracy |
| LLM10 | Model Theft | The model is an asset, and the API is a disclosure surface |

## The three chains that connect them

**Chain 1, the injection chain.** Prompt injection (LLM01) is the entry. Improper output handling (LLM02) and excessive agency (LLM08) decide the blast radius. Insecure plugins (LLM07) are the mechanism by which agency is exercised. *Injection is inevitable; impact is a design choice.*

**Chain 2, the integrity chain.** Supply chain compromise (LLM05) and training data poisoning (LLM03) both corrupt the model before it runs. Both are invisible to clean accuracy testing. Both are fixed by provenance, not by scanning.

**Chain 3, the trust chain.** Sensitive information disclosure (LLM06) and overreliance (LLM09) are what happens when humans and systems trust output they should verify. Model theft (LLM10) is the same trust problem pointed at the model itself as an asset.

## Corrections and updates to the course material

| Course position | Current position |
| --------------- | ---------------- |
| The 2023 numbering (LLM01 to LLM10) | The **2025 (v2.0)** list, published 18 Nov 2024, renumbers everything. **Insecure Plugin Design, Overreliance and Model Theft no longer exist as standalone entries.** See the introduction for the full mapping |
| Insecure Output Handling is second highest | It is **LLM05 Improper Output Handling** in 2025; LLM02 is now Sensitive Information Disclosure |
| Model Denial of Service | **LLM10 Unbounded Consumption**, broadened to include cost as well as availability |
| Training Data Poisoning | **LLM04 Data and Model Poisoning**, broadened beyond data |
| Lee Luda regulatory action | The PIPC fined ScatterLab **KRW 103.3 million in April 2021**, not 2023. It predates the ChatGPT era and was the first Korean application of data protection law to an AI system |
| Plugins as the integration mechanism | Plugins became GPTs, then Actions, and the industry converged on **MCP**. The attack class is unchanged, but MCP added tool poisoning, rug pulls, cross tenant leaks and real CVEs including an RCE at CVSS 9.6 |
| Mobile model extraction study | The stronger evidence is **"Mind Your Weight(s)"** (USENIX Security 2021, arXiv 2002.07687): 46,753 apps analysed, 1,468 on device ML apps, **41% storing models in plaintext** |

## What the chapter does not cover, and should

1. **System Prompt Leakage and Vector and Embedding Weaknesses**, the two new 2025 entries, both driven by the shift to RAG.
2. **MCP and agentic security**, which is where plugin design and excessive agency actually live in 2026: tool poisoning, rug pulls, toxic agent flows, MCP gateways.
3. **The lethal trifecta** as a design test: private data, untrusted content, outbound channel. An agent with all three will leak. Remove one.
4. **Automated verification of checkable claims**, which converts overreliance from a human discipline problem into a deterministic control.

## The five ideas worth keeping

1. **Injection is the way in; output handling and agency decide how bad it gets.** You cannot fully prevent the first. You fully control the second and third.
2. **A model cannot police itself.** Its instructions are attacker influenceable, so authorisation belongs in deterministic downstream systems.
3. **Clean accuracy detects neither poisoning nor backdoors.** Integrity requires provenance, not testing.
4. **Fluency is constant regardless of accuracy.** The output gives the reader no signal, which is why verification must be structural rather than left to judgement.
5. **Every capability you grant is a capability an injection inherits.** Least privilege is the highest leverage control in the entire list.
