# The Threat Model Parlance

Shared vocabulary, so that a threat model session does not stall on definitions.

## The Microsoft SDL threat modeling workflow

Five steps, confirmed against Microsoft SDL documentation:

1. **Define security requirements.** The team defines requirements according to customer and business needs.
2. **Create an application diagram.** Visualise the architecture and data flows.
3. **Identify threats.** Development and security teams elicit threats against the diagram.
4. **Mitigate threats.** Address the issues in the system design.
5. **Validate that threats have been mitigated.** Confirm the controls work, then analyse and report the state of the project.

Threat modeling should be part of the routine development lifecycle, refined on each pass to further reduce risk.

## Assets

**What are we trying to protect?** If a thing has no value, protecting it is wasted effort. Assets fall into three categories, and the third is the one people forget:

1. **Tangible things attackers want.** Social security numbers, PII, secrets, credentials, intellectual property, model weights.
2. **Things we want to protect.** Customer trust, reputation, goodwill. Not stored anywhere, but destroyed by a breach.
3. **Stepping stones with no immediate value.** A machine inside the VPN with internet access. An empty safe. The safe holds nothing today, but leaving an attacker free to probe it is still a problem, because tomorrow it holds a million dollars, and because the probing teaches the attacker how the lock works.

For AI systems the asset list expands: training data, fine tuning data, model weights, system prompts, embeddings and vector stores, conversation history, and the inference API itself.

## Weakness

**A type of mistake in software that, under the right conditions, could contribute to the introduction of a vulnerability.** The term applies regardless of whether the mistake occurs in implementation, design or any other SDLC phase.

Examples: missing authorization, unsafe deserialisation, out of bounds read. This is what the CWE catalogue enumerates.

## Vulnerability

**An occurrence of a weakness (or several) within software, where the weakness can be used by a party to cause the software to modify or access unintended data, interrupt proper execution, or perform actions not granted to that party.**

The distinction that matters: a **weakness is a class of mistake**; a **vulnerability is an instance of that mistake in a specific system, that someone can actually use.** CWE catalogues weaknesses; CVE catalogues vulnerabilities.

Example: improper neutralisation of special elements in an OS command (the weakness) becomes remote code execution (the vulnerability) in a specific product. Unsafe deserialisation of a malicious object does the same, which is precisely the risk in loading pickle based model files.

## Threat versus risk

A **threat** is a potential negative action against an asset. A **risk** is that threat combined with its likelihood and impact. Every threat poses a risk, but not every threat poses a risk worth treating. This distinction is what the "Rating and Managing Risks" lesson depends on.
