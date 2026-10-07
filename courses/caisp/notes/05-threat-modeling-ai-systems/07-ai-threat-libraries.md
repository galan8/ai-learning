# AI Threat Libraries

## Why libraries are needed

The goal of threat modeling is to secure a system: identify threats, rank them, and test that mitigations work. This lesson focuses only on **identification**.

STRIDE applies to software but **is not a threat library**. Faced with an LLM application, STRIDE will prompt you to ask about spoofing and tampering, but it will not tell you that prompt injection, training data poisoning, model extraction or excessive agency exist. For AI systems, STRIDE alone leaves you staring at a blank page. Threat libraries fill that gap by supplying the catalogue of things that have actually gone wrong.

## OWASP Top 10 for LLM Applications

Ten threats with example incidents, plus published example architectures showing where each risk applies. The 2023 and 2025 versions differ in numbering, so always state which you are citing. Covered in detail in the Chapter 3 notes.

## MITRE ATLAS

**Adversarial Threat Landscape for Artificial Intelligence Systems.** Modelled on MITRE ATT&CK: tactics are the adversary's goals, techniques are how they achieve them, and case studies map real incidents to them.

### Correction to the course material

The course lists **14 tactics**. ATLAS has since expanded to **16**, and has systematically renamed "ML" to "AI" across tactics, techniques and mitigations. As of the November 2025 release (v5.1.0) it documents 16 tactics, 84 techniques, 32 mitigations and 42 case studies, and it continues to grow, so cite the version and date.

The current matrix order:

| # | Tactic | ID | Note |
| - | ------ | -- | ---- |
| 1 | Reconnaissance | AML.TA0002 | |
| 2 | Resource Development | AML.TA0003 | |
| 3 | Initial Access | AML.TA0004 | |
| 4 | **AI Model Access** | AML.TA0000 | renamed from ML Model Access |
| 5 | Execution | AML.TA0005 | |
| 6 | Persistence | AML.TA0006 | |
| 7 | Privilege Escalation | AML.TA0012 | |
| 8 | Defense Evasion | AML.TA0007 | |
| 9 | Credential Access | AML.TA0013 | |
| 10 | Discovery | AML.TA0008 | |
| 11 | **Lateral Movement** | AML.TA0015 | added Oct 2025 |
| 12 | Collection | AML.TA0009 | |
| 13 | **AI Attack Staging** | AML.TA0001 | renamed from ML Attack Staging |
| 14 | **Command and Control** | AML.TA0014 | added Apr 2024 |
| 15 | Exfiltration | AML.TA0010 | |
| 16 | Impact | AML.TA0011 | |

### Real incidents mapped to tactics, with the control that would have prevented them

| Tactic | Documented case | What happened | Fundamental control |
| ------ | --------------- | ------------- | ------------------- |
| Reconnaissance / Defense Evasion | **Cylance bypass** (AML.CS0003) | Skylight Cyber reverse engineered the product and found a universal string that flipped malware scores to benign | Limit public release of model detail (M0000); adversarial input detection (M0015); ensemble detection so one model's judgement is not the only gate |
| AI Model Access | **Clearview AI misconfiguration** (AML.CS0006) | spiderSilk found a misconfigured source repository allowing open account creation, exposing models and algorithms | Control access to AI models and data at rest (M0005); conventional access control hardening |
| Execution | **MathGPT code execution** (AML.CS0016) | Prompt injection into a natural language to Python service achieved arbitrary code execution and exposed the GPT-3 API key | Generative AI guardrails (M0020); sandboxed execution with no credentials; never pass model output to eval |
| Persistence | **Tay poisoning** (AML.CS0009) | Microsoft's chatbot learned from Twitter interactions and was coordinated into offensive output within 24 hours | Sanitise training data (M0007); do not learn from unvetted user input; telemetry logging (M0024) |
| Persistence / Supply Chain | **PoisonGPT** (AML.CS0019) | Mithril Security edited one fact into GPT-J-6B with ROME and uploaded it under a typosquatted org name | Dataset and model provenance (M0025); validate AI model (M0008); verify AI artifacts (M0014) |
| Initial Access / Impact | **Camera hijack on facial recognition** (AML.CS0004) | A hijacked camera feed defeated facial recognition authentication in a tax fraud scheme | Liveness and channel integrity checks; multi factor authentication that does not rely on one biometric signal |
| Initial Access / Impact | **ShadowRay** (AML.CS0023) | Oligo documented in the wild exploitation of exposed Ray Jobs APIs on AI clusters | Authentication on all management APIs; network segmentation; never expose orchestration planes |
| Credential Access | **Hugging Face token exposure** (Lasso research) | 1,681 valid tokens found, 655 with write access to major model repositories | Secret scanning in code and CI; scoped read only tokens; rotation |
| Defense Evasion / AI Attack Staging | **ProofPoint evasion** (AML.CS0008, CVE-2019-20634) | Leaked scores in mail headers let researchers train a proxy model and craft evading messages | Passive output obfuscation (M0002): do not leak confidence scores; restrict query volume (M0004) |
| AI Attack Staging / Impact | **VirusTotal poisoning** (AML.CS0002) | Adversarial samples submitted to poison a shared detector | Provenance and vetting of shared training corpora; anomaly detection on submissions |
| Exfiltration | **ChatGPT plugin privacy leak** (AML.CS0021) | Johann Rehberger showed indirect prompt injection from a malicious page hijacking a session and exfiltrating conversation history | Guardrails (M0020); restrict tool invocation on untrusted content; egress allowlisting |
| Collection / Impact | **Morris II worm** (AML.CS0024) | A zero click self replicating GenAI worm propagating through RAG email context | Human in the loop for agent actions (M0029); treat retrieved content as untrusted; least privilege on mail tools |
| Privilege Escalation / Impact | **M365 Copilot hijack** (AML.CS0026) | Zenity red team demonstrated manipulation of an assistant toward financial actions | Privileged agent permission configuration (M0026); human approval for consequential actions (M0029) |
| Command and Control | **SesameOp** (AML.CS0042) | Microsoft DART found an implant using the OpenAI Assistants API as a covert C2 channel | Egress monitoring and allowlisting; treat AI API traffic as a monitored channel, not an implicitly trusted one |
| Exfiltration (model theft) | **GPT-2 replication** (AML.CS0007) | Researchers replicated a model from published detail and query access | Limit model artifact release (M0001); rate limit queries (M0004) |

Around 70% of ATLAS mitigations map to security controls organisations already run, which makes adoption a mapping exercise rather than a rebuild.

## The AI Threat Mind Map

By **Sandy Dunn** (GitHub subzer0girl2), an OWASP LLM Top 10 contributor. Its stated purpose is to help defenders understand the different types of threat and implement appropriate controls. Version 2.0 has **seven top level branches**:

```
                        ┌───────────────────────────┐
                        │ Threat NOT understanding  │
                        │ AI models                 │
                        └─────────────▲─────────────┘
   ┌────────────────────┐             │            ┌──────────────────────┐
   │ Threat of AI       │◄──────┐     │      ┌────►│ Threat FROM AI       │
   │ dependency         │       │     │      │     │ models               │
   └────────────────────┘       │     │      │     └──────────────────────┘
                            ┌───┴─────┴──────┴───┐
   ┌────────────────────┐   │                    │  ┌──────────────────────┐
   │ Threats NOT using  │◄──┤   AI THREAT MAP    ├─►│ Threat USING AI      │
   │ AI models          │   │       v2.0         │  │ models               │
   └────────────────────┘   │                    │  └──────────────────────┘
                            └───┬────────────┬───┘
   ┌────────────────────┐       │            │     ┌──────────────────────┐
   │ AI legal and       │◄──────┘            └────►│ Threat TO AI models  │
   │ regulatory threat  │                          │                      │
   └────────────────────┘                          └──────────────────────┘
```

Reading each branch as a distinct question, which is what makes the map useful:

1. **Threat TO AI models.** Attacks against your model: poisoning, evasion, extraction, backdoors. This is where ATLAS and the OWASP LLM Top 10 mostly sit.
2. **Threat FROM AI models.** Harm the model itself causes: hallucination, bias, unsafe output, hazardous advice. No attacker required.
3. **Threat USING AI models.** Adversaries using AI as a capability: deepfake phishing, malware generation, WormGPT and FraudGPT.
4. **Threats NOT using AI models.** Conventional attacks against the infrastructure hosting AI: exposed APIs, stolen credentials, misconfigured storage. ShadowRay and the Hugging Face token exposure live here, and this branch is easy to forget precisely because it is not exotic.
5. **Threat of AI dependency.** Business risk from relying on AI: vendor lock in, model deprecation, availability of a third party service, skill atrophy.
6. **Threat NOT understanding AI models.** Risk from deploying what you do not comprehend: unclear provenance, no interpretability, misplaced confidence in outputs.
7. **AI legal and regulatory threat.** EU AI Act, copyright in training data, privacy law, sector regulation, and liability for what your system says (see the Air Canada ruling in the Chapter 3 notes).

The branches 3 through 7 are the value of this library: they cover ground that neither STRIDE nor ATLAS reaches.

## MIT AI Risk Repository

### Correction to the course material

The course cites 1,600 risks. The figures have moved: **777 risks at launch (August 2024), 1,612 classified risks (April 2025), and over 1,700 across 74 frameworks by December 2025.** Cite it as a dated series. Run by MIT FutureTech.

It contains **two taxonomies**, and the course only covers one:

**Causal taxonomy** (how a risk arises), three factors:
1. **Entity:** human, AI, or other
2. **Intentionality:** intentional, unintentional, or other
3. **Timing:** pre deployment, post deployment, or other

**Domain taxonomy:** **7 domains and 24 subdomains** (the course says 23; a multi agent subdomain was added in April 2025):
1. Discrimination and toxicity
2. Privacy and security
3. Misinformation
4. Malicious actors and misuse
5. Human computer interaction
6. Socioeconomic and environmental
7. AI system safety, failures and limitations

The subdomains under domain 7, which the course lists: AI pursuing its own goals in conflict with human goals or values; AI possessing dangerous capabilities; lack of capability robustness; lack of transparency or interpretability; AI welfare and rights; and multi agent risks.

The causal taxonomy is the more useful half for threat modeling, because "unintentional, pre deployment, human" and "intentional, post deployment, AI" call for entirely different controls.

## BIML Machine Learning Risk Framework

### Correction to the course material

The course says 8 components. The Berryville Institute of Machine Learning's *Architectural Risk Analysis of Machine Learning Systems* (January 2020, Gary McGraw and colleagues) identifies **78 risks across nine components**, plus a top 10 and a set of system wide risks. BIML also published an *Architectural Risk Analysis of Large Language Models* (2024) identifying **81 LLM risks**, highlighting a top 10 and detailing 23 black box foundation model risks.

BIML is oriented toward data scientists and ML engineers rather than security teams, and its component by component structure (data, training, evaluation, input, model, inference, output and so on) maps neatly onto a DFD, which makes it the most directly usable library when the diagram is already drawn.

## AI Incident Database

Run by the **Responsible AI Collaborative**, modelled on aviation and computer security incident databases. It indexes real world harms from deployed AI systems: over 1,200 reports and growing, with the annual rate rising sharply (233 incidents recorded in 2024, 362 in 2025 per the Stanford AI Index).

Its role in threat modeling is evidence. When someone asks whether a threat is realistic, the database answers with what has actually happened to somebody.

## Related paper

**"The Risks of Machine Learning Systems"** by Samson Tan, Araz Taeihagh and Kathy Baxter (arXiv:2204.09852, April 2022). Proposes the Machine Learning System Risk framework, dividing risks into **first order** (arising from the system's design and implementation choices) and **second order** (consequences that appear when the system meets the world). It reasons about ML *systems* rather than technologies, which is the right unit of analysis for threat modeling.

## Frameworks the course does not cover

Worth knowing, because several are now what regulators and enterprises actually ask for.

| Framework | What it is | When to use it |
| --------- | ---------- | -------------- |
| **NIST AI RMF (AI 100-1)** | Voluntary risk management framework, four functions: **Govern, Map, Measure, Manage**, with a Playbook and a Generative AI Profile (AI 600-1) | Enterprise AI governance and a shared vocabulary with risk and compliance functions |
| **NIST AI 100-2** | Adversarial ML taxonomy: evasion, poisoning, privacy and abuse; the 2025 edition extends to LLMs, RAG and agents | The authoritative attack taxonomy to sit alongside ATLAS |
| **Google SAIF** | Secure AI Framework; six elements across data, infrastructure, model and application; SAIF 2.0 adds agent guidance | Lifecycle wide control coverage |
| **Databricks AI Security Framework (DASF)** | v3.0 covers 13 AI system components with mapped risks and controls, including agentic AI | Hands on control selection across the ML lifecycle |
| **ENISA AI Threat Landscape** | EU asset based threat taxonomy across the ML lifecycle, strong on supply chain | EU regulatory alignment |
| **PLOT4AI** | Threat library and card deck, 100+ threats across 8 domains including ethics, privacy and non compliance | Privacy, safety and ethics by design workshops |
| **STRIDE-AI** | Extends STRIDE to ML assets (Mauri and Damiani); note a separate 2026 GenAI framework shares the name | Adapting classic STRIDE to ML failure modes |
| **MAESTRO** | Cloud Security Alliance, seven layer threat modeling for agentic AI: foundation models, data operations, agent frameworks, deployment infrastructure, evaluation and observability, security and compliance, agent ecosystem | Autonomous and multi agent systems, where STRIDE and ATLAS both run short |
| **Microsoft "Threat Modeling AI/ML Systems and Dependencies"** | SDL supplement built on the Failure Modes in Machine Learning taxonomy | Teams already using Microsoft SDL |

**Selection guidance:** NIST AI RMF for governance, ATLAS and NIST AI 100-2 for the attack taxonomy, BIML or DASF for component level controls, PLOT4AI when privacy and ethics are in scope, and MAESTRO once agents and tools enter the architecture.
