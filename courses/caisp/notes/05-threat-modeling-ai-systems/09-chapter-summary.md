# Chapter 5 Summary

## The argument in one line

Threat modeling is the only security technique that finds design flaws before code exists; applied to AI systems it needs a diagram that includes training data and model artifacts, a threat library that knows AI attacks exist, and a rating method that turns findings into an order of work.

## The chain

1. **What is threat modeling.** Analysing a representation of a system to find security and privacy concerns, without running code. Four questions: what are we working on, what can go wrong, what are we going to do about it, did we do a good enough job.
2. **Why.** It is the only technique available before the system exists, and design flaws are the most expensive defects to fix later. Its main challenge is staying in sync with a changing design.
3. **Parlance.** Assets are what has value, including stepping stones. A weakness is a class of mistake; a vulnerability is an exploitable instance of it. A threat becomes a risk when you attach likelihood and impact.
4. **Diagramming.** The DFD, deliberately stripped of control flow, with five elements and trust boundaries. Attacks follow data, so boundary crossings are where threats concentrate.
5. **Architecture.** A reference LLM system across four trust zones. Real systems add RAG stores, plugins, system prompts and third party APIs.
6. **STRIDE.** Six categories, each mapping to a security property, applied against the architecture to elicit threats. A method of discovery, not a filing system.
7. **Threat libraries.** STRIDE will not tell you prompt injection exists. ATLAS, the OWASP LLM Top 10, the AI Threat Mind Map, MIT's repository, BIML and the AI Incident Database supply the AI specific catalogue.
8. **Rating and managing.** OWASP factor scoring, averaged and banded, combined in a severity matrix, then treated by avoiding, mitigating, accepting or transferring.

## Corrections to the course material

| Course says | Current position |
| ----------- | ---------------- |
| MITRE ATLAS has 14 tactics | **16 tactics**; ML renamed to AI throughout; Command and Control (AML.TA0014) and Lateral Movement (AML.TA0015) added. 84 techniques, 32 mitigations, 42 case studies as of v5.1.0, Nov 2025 |
| MIT AI Risk Repository has 1,600 risks | **777 at launch (Aug 2024), 1,612 (Apr 2025), 1,700+ (Dec 2025)** across 74 frameworks. **7 domains and 24 subdomains**, not 23 |
| BIML has 8 components and 78 risks | **Nine components, 78 risks** (2020); plus **81 LLM risks** in the 2024 LLM analysis |
| Risk band: 0 to less than 4 is low | **0 to under 3 is LOW**, 3 to under 6 MEDIUM, 6 to 9 HIGH |
| OWASP risk is likelihood × impact | Factors are **averaged** within likelihood and impact, banded, then combined in a **severity matrix** |

## What the chapter does not cover, and should

1. **The causal taxonomy** in the MIT repository (entity, intentionality, timing), which is more useful for threat modeling than the domain taxonomy alone.
2. **NIST AI RMF and NIST AI 100-2**, now the common language with regulators and enterprise risk functions.
3. **MAESTRO**, the seven layer framework for agentic AI, which matters the moment tools and autonomy enter the architecture.
4. **Agentic and multi agent threats** generally. ATLAS added Lateral Movement and Command and Control precisely because agents changed the threat surface.
5. **Threat modeling as code**, which is the practical answer to the "keeping models in sync" challenge the chapter raises but does not resolve.

## The three ideas worth keeping

**Attacks follow data.** That is why the DFD works, why trust boundaries are the high yield part of the diagram, and why the AI specific additions (training data, RAG sources, model artifacts) are the ones that matter.

**STRIDE finds categories; libraries find AI threats.** Use both. STRIDE alone on an LLM produces a generic model. A library alone produces a checklist detached from your architecture.

**Rating is about making judgement explicit, not about precision.** The numbers are a language for disagreement. Their purpose is to produce an ordered list that a team can defend and act on.
