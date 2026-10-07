# STRIDE Threat Examples for LLM Applications

## What STRIDE is

STRIDE is a mnemonic for eliciting threats, created by **Loren Kohnfelder and Praerit Garg at Microsoft in the paper "The Threats to Our Products", 1 April 1999.**

Each letter names a threat category and the security property it violates:

| STRIDE | Property violated | Security mechanism |
| ------ | ----------------- | ------------------ |
| **S**poofing | Authenticity | Authentication |
| **T**ampering | Integrity | Integrity controls |
| **R**epudiation | Non-repudiation | Logging and audit |
| **I**nformation Disclosure | Confidentiality | Confidentiality controls |
| **D**enial of Service | Availability | Availability controls |
| **E**levation of Privilege | Authorization | Authorization |

**STRIDE is a method of identifying threats, not merely a classification taxonomy.** The categories interact: an attacker might tamper with a file in order to spoof an identity. Do not spend session time arguing which letter a threat belongs to; the letters are prompts for discovery, not filing cabinets.

## Applied to the reference architecture

### Spoofing

1. An attacker spoofs a legitimate user by bypassing authentication or authorization at the gateway
2. Feeding misleading input to the AI system, for example a crafted prompt to a chatbot or a presented artifact to a facial recognition system
3. Deploying a fake version of the model and interacting with it as though it were legitimate, or presenting a fake model to users

**Controls:** strong authentication and MFA at the gateway, signed and scoped API keys, binding the agent's identity to the user's, mutual TLS between internal services, and verifying model and plugin provenance by signature and pinned revision.

### Tampering

1. Tampering with training data used to build the model, producing poor or malicious decisions
2. Injecting malicious data into the trained model to steer a specific outcome
3. API manipulation: modifying input or code to alter system behaviour

**Controls:** dataset provenance and integrity verification (ATLAS AML.M0025 Maintain AI Dataset Provenance), training data sanitisation (AML.M0007), model validation before promotion (AML.M0008), artifact verification and signing (AML.M0014), write access control on training data stores, and safe serialisation formats so loading a model cannot execute code.

### Repudiation

1. An attacker deletes or tampers with logs to deny that data was manipulated
2. Altering records of interactions to deny actions taken, making auditing and attribution difficult
3. Erasing logs to prevent accountability, or manipulating legal or financial decisions without trace

**Controls:** immutable and append only logging, AI telemetry logging covering prompts, responses and tool calls (AML.M0024), log shipping to a separate trust zone, cryptographic signing of audit records, and strict separation of duties so the identity that can act cannot also edit the log.

### Information Disclosure

1. The model exposes personal or sensitive information such as identifiers or private details
2. An attacker extracts private details from the model, or steals the model itself by querying it repeatedly (model extraction)
3. Gaining access to proprietary information or sensitive data stores

**Controls:** output filtering and guardrails (AML.M0020), PII redaction before storage and before display, least privilege on RAG retrieval so users cannot retrieve documents they are not entitled to, encryption at rest and in transit, query rate limits to make extraction expensive (AML.M0004), and passive output obfuscation such as withholding confidence scores (AML.M0002).

### Denial of Service

1. Flooding the AI service with excessive requests, making it slow or unavailable
2. Resource exhaustion that crashes the service or prevents the model from being trained properly

**Controls:** rate limiting per identity and per IP, token and cost budgets, input size and output length caps, timeouts and queue limits, separate quotas for expensive operations, and autoscaling with circuit breakers. See the Chapter 3 notes on unbounded consumption: on metered inference the first symptom is often a bill rather than an outage.

### Elevation of Privilege

1. Exploiting a weakness in the API gateway to reach unauthorised features
2. The model performing actions beyond the user's entitlement, typically through tools or plugins

**Controls:** least privilege on agent and tool permissions (AML.M0026), human in the loop approval for consequential actions (AML.M0029), restricting tool invocation when the context contains untrusted data, sandboxing any code execution with no credentials and no network, and enforcing authorization at the resource rather than trusting the model to respect it.

## The caveat the lesson gives, which is correct

Your threats will not match this list exactly. STRIDE applied to your architecture, by people who understand it, produces a different and better set than any generic list. Use these as priming examples, not as a checklist to copy.

## The limitation worth naming

STRIDE was designed for software systems and is not itself a threat library. Applied to an LLM, it will prompt you toward the six categories but will not tell you that prompt injection, training data poisoning or model extraction exist. For those you need a threat library, which is the subject of the next lesson.
