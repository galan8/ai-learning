# Exercise: Threat Modeling an AI System

Course: CAISP (Practical DevSecOps)
Status: Complete (all six steps)

## How to read this document

This is the hands-on companion to the Chapter 5 notes. It is a paper exercise, no code, so it does not use the two-audience split. It walks the full threat modeling workflow on one AI architecture, then flags where the lab's method differs from the notes.

---

## 1. What we are doing

We take a single AI connected system, drawn as a data flow diagram, and work the complete threat modeling loop against it: set the scope, define security requirements, diagram the system, elicit threats with STRIDE, and rate the resulting risks. This is Chapter 5's theory applied end to end to one concrete architecture.

The point is not the specific threats we find. It is the **method**: a repeatable way to reason about what can go wrong in a system before it is built or attacked.

---

## 2. Step one: scope

The first two rules of threat modeling, from the lab:

1. Choose a system to threat model.
2. Identify the **scope** of the activity, meaning how much of the system, and how much time.

Scope is what separates traditional from agile threat modeling, and it is worth restating because it is the single most practical decision in the whole exercise:

| | Traditional | Agile / DevOps |
| - | ----------- | -------------- |
| Scope | An entire system, for a big release | One use case or feature, for a sprint |
| Who | Expert threat modelers | The delivery team |
| Time | Weeks | Hours |
| Output | Documents | Design improvements in the issue tracker |

This lab deliberately chooses the **traditional** scope: the entire AI connected system. That is the right choice for learning (you see everything), but the Chapter 5 notes' warning applies: a whole-system model is the one that goes stale fastest. In practice you would scope to a feature and keep the model in version control beside the design.

---

## 3. Step two: security requirements

Requirements and threats feed each other, and this bidirectional relationship is the subtle idea in this step:

- **Requirements drive mitigations.** A compliance rule ("no injection attacks") becomes a requirement ("implement input validation"), which becomes a control.
- **Threats drive requirements.** Threat modeling surfaces threats whose mitigations become new requirements.
- **Requirements lower risk scores.** If input validation is already a requirement, a SQL injection threat is *less likely* to materialise, so it scores lower. Existing controls feed directly into the likelihood calculation later.

Where requirements come from (the lab's list, worth knowing as a menu): OWASP ASVS Level 1, MITRE CAPEC high-rated threats, STRIDE's security properties, the OWASP Top 10, PCI DSS requirement 6, IEEE EAD, and **NIST's AI Risk Management Framework**. That last one is the AI-specific entry and connects to the frameworks section of the Chapter 5 notes.

For this exercise, the requirements are simply STRIDE's six security properties, which is the cleanest possible statement of "defend against each threat category":

| Security property | Defends against |
| ----------------- | --------------- |
| Authentication | Spoofing |
| Integrity | Tampering |
| Non-repudiation | Repudiation |
| Confidentiality | Information Disclosure |
| Availability | Denial of Service |
| Authorization | Elevation of Privilege |

This table is worth memorising: it is the STRIDE-to-property mapping from the notes, and it doubles as the requirements list.

---

## 4. Step three: the diagram

The architecture, transcribed from the DFD, across four trust zones:

```
   ┌────────┐        ┌──────── DMZ ─────────┐   ┌── Internal: Interface ──┐
   │  User  │        │                      │   │      to DMZ             │
   │(entity)│        │  ╭────────────────╮  │   │                         │
   └───┬────┘        │  │   Security      │◄─┼───┼──╮                      │
       │             │  │   Authentication│  │   │  │  ╭──────────────╮    │
       ▼             │  │   System        │──┼───┼──┼─►│  AI Model    │◄───┼──┐
   ╭────────╮        │  │  (AuthN + AC)   │  │   │  │  │ (Inference)  │    │  │
   │Browser │◄───────┼──┤                 │  │   │  ╰──╰──┬────┬──────╯    │  │
   │  /App  │───────►│  ╰──▲──────────┬───╯  │   └───────┼────┼───────────┘  │
   ╰────────╯        │     │          │      │           │    │              │
                     │  ╭──┴──────────▼───╮  │   ┌── Internal: Protected ──┐ │
                     │  │  API Gateway /  │  │   │   ═══════════════       │ │
                     │  │  Proxy (Routing)│  │   │   │ Data Layer  │◄──────┼─┘ (from AI Model)
                     │  ╰─────────────────╯  │   │   │ (User Data) │       │ │
                     └──────────────────────┘   │   ═══════════════       │ │
                                                │   ═══════════════       │ │
                                                │   │    Logs     │◄──────┼─┘ (from AI Model)
                                                │   ═══════════════       │
                                                └─────────────────────────┘

   ┌──────── Internal Network: Development ────────┐
   │   ═══════════════        ╭─────────────────╮  │
   │   │ Training    │───────►│ Model Training  │──┼──► AI Model (Inference)
   │   │ Data        │        │ Pipeline        │  │
   │   ═══════════════        ╰─────────────────╯  │
   └───────────────────────────────────────────────┘

   Legend:  ╭──╮ process   │ │ external entity   ═══ data store
```

**The four trust zones, and why the boundaries matter:**

1. **DMZ (public facing):** API Gateway/Proxy and the Security Authentication System. First contact, so spoofing and denial of service concentrate here.
2. **Internal, interface to DMZ:** the AI Model doing inference. Receives data that originated outside, so this is where prompt-borne threats reach.
3. **Internal, protected:** Data Layer and Logs. Information disclosure and repudiation concentrate here.
4. **Internal, development:** Training Data and the Training Pipeline. Tampering concentrates here, and note the arrow: this zone **writes into the production inference model**, so a compromise here propagates forward.

The data flows: user request goes Browser → Gateway → Auth → AI Model → (Data Layer, Logs); training goes Training Data → Pipeline → AI Model. The existing controls the diagram encodes: DMZ isolation, authentication at the gateway, a protected network for sensitive data, and a separate development environment.

**The Chapter 5 caveat still applies:** this diagram omits the components that carry most real risk in a modern LLM deployment (RAG stores and vector databases, plugins and tools, the system prompt, third-party model APIs). It is a good teaching baseline, not a production-complete model.

---

## 5. Step four: eliciting threats with STRIDE

STRIDE is a discovery aid, not a taxonomy. The lab is refreshingly honest about this, and it matches the notes: the example threats "are not classified correctly" because **STRIDE is a mnemonic for finding threats, not a scheme for filing them.** Do not spend session time arguing which letter a threat belongs to.

The elicitation prompts, per category, applied to this architecture:

### Spoofing (authenticity)

1. **An attacker spoofs the user's identity**, by impersonating their email or device.
2. **An adversary deploys a fake version of the AI model**, so users interact with an impostor, causing misinformation or loss of trust.

### Tampering (integrity)

1. **Tampering with the training data**, introducing bias or false information, so the model makes poor or malicious decisions. (The development zone writing into production is what makes this severe.)
2. **Injecting malicious data into the trained model** to skew its outputs.

### Repudiation (non-repudiation)

1. **Deleting logs** to deny having tampered with training data.
2. **Altering logs** to erase evidence of model misuse, defeating accountability for harmful automated decisions.

### Information Disclosure (confidentiality)

1. **The model inadvertently discloses confidential data** (an SSN, health information).
2. **Reverse engineering the model** by querying it repeatedly, recovering proprietary logic or training data. (This is model extraction and model inversion from the Chapter 6 notes.)

### Denial of Service (availability)

1. **Flooding the service with excessive requests**, making it slow or unavailable.
2. **Overloading training resources**, crashing training or preventing the model from training properly.

### Elevation of Privilege (authorization)

1. **Exploiting the API Gateway or Auth system** to reach unauthorised features of the AI system or training pipeline.
2. **Exploiting the model itself** to gain higher access within the system.

**Every STRIDE category produced at least one threat**, which is the lab's stated bar and a good discipline: it forces you to consider all six lenses rather than stopping at the obvious ones.

---

## 6. Step five: rating the risks

The lab uses a **simplified** risk method: rate likelihood and impact each on a 1 to 10 scale, then **average them**.

```
   Risk = (Likelihood + Impact) / 2
```

| Band | Value |
| ---- | ----- |
| Critical | 10 |
| High | 7 to 9 |
| Medium | 4 to 6 |
| Low | 1 to 3 |

The ranked results:

| Threat | L + I | Risk | Band |
| ------ | ----- | ---- | ---- |
| Spoof the user's identity | 10 + 10 | 10 | **Critical** |
| Disclose confidential information | 10 + 10 | 10 | **Critical** |
| Flood the service (DoS) | 10 + 10 | 10 | **Critical** |
| Tamper with training data | 8 + 10 | 9 | High |
| Reverse engineer the model | 7 + 7 | 7 | High |
| Deploy a fake model | 2 + 10 | 6 | Medium |
| Inject malicious data into the model | 5 + 7 | 6 | Medium |
| Overload training resources | 2 + 10 | 6 | Medium |
| Unauthorized access to AI system / pipeline | 2 + 10 | 6 | Medium |
| Gain higher access within the system | 2 + 10 | 6 | Medium |
| Delete logs | 3 + 7 | 5 | Medium |
| Alter logs | 3 + 7 | 5 | Medium |

**How to read this ranking:** the three criticals are all things with maximum likelihood *and* maximum impact, and notably they are the most conventional threats (identity spoofing, data disclosure, DoS), not the exotic AI-specific ones. The AI-specific threats (fake model, training overload, model reverse engineering) mostly score Medium, because their **likelihood is low**: they require access to internal resources an external attacker rarely has. Impact is high, but likelihood pulls the score down. That is the risk method doing its job: separating "catastrophic but improbable" from "catastrophic and easy".

Note the rounding rule the lab uses: 7.5 rounds up to 8. Minor, but worth recording for reproducibility.

---

## 7. Where this lab diverges from the Chapter 5 notes

The exercise uses a **deliberately simplified** risk method, and it is worth being explicit about how it differs from the OWASP Risk Rating Methodology documented in the notes, because the two give different answers.

| | This lab | OWASP Risk Rating (Ch5 notes) |
| - | -------- | ----------------------------- |
| Likelihood | One number, 1 to 10, by judgement | Averaged from 8 factors (skill, motive, opportunity, size, ease of discovery, ease of exploit, awareness, intrusion detection) |
| Impact | One number, 1 to 10, by judgement | Averaged from 8 factors (technical: confidentiality, integrity, availability, accountability; business: financial, reputation, non-compliance, privacy) |
| Combination | **Average** of L and I | **Matrix** of L band vs I band |
| Low band | 1 to 3 | 0 to under 3 |

**The averaging is the meaningful difference, and it hides a real flaw.** Averaging lets a high value on one axis be dragged down by a low value on the other. "Deploy a fake model" scores impact 10 and likelihood 2, averaging to 6 (Medium). But a threat that is *devastating* if it happens should arguably not be filed next to genuinely middling risks just because it is currently improbable, especially when "improbable" rests on an assumption (that an attacker cannot reach internal resources) that a single other breach could overturn. OWASP's matrix keeps the two axes visible rather than collapsing them, which is why the notes prefer it and why OWASP itself recommends more mature methods for serious work.

The practical takeaway: a simple averaging method is fine for a fast, teachable pass, but know that it **systematically under-weights low-likelihood high-impact threats**, which for AI systems (model theft, training poisoning) are exactly the ones you most need to see clearly.

---

## 8. What comes after rating, which the lab notes but does not do

The loop is not finished at a ranked list. The lab correctly states the next move: **the identified risks and their mitigations must be converted into requirements a development team can consume.** A threat model that ends in a document nobody builds from has produced nothing. In an agile setting that means issues in the tracker with owners, feeding back into the requirements step and closing the loop the Chapter 5 notes describe (risk analysis informs the engineering story, which triggers threat modeling, which mitigates).

---

## 9. Summary

1. Threat modeling is a repeatable loop: scope, requirements, diagram, elicit, rate, and convert to requirements.
2. Scope is the key decision; this lab uses whole-system (traditional) scope, which teaches well but goes stale fast.
3. Requirements and threats feed each other: existing controls lower likelihood scores.
4. The DFD's four trust zones are where threats concentrate, and the development-writes-to-production arrow is what makes training tampering severe.
5. STRIDE finds threats; it does not classify them, and every category should yield at least one.
6. The three critical risks are conventional (spoofing, disclosure, DoS); the AI-specific threats score lower on likelihood, not impact.
7. The lab's averaging risk method is simpler than the OWASP matrix in the notes, and systematically under-weights the low-likelihood high-impact AI threats that matter most.
8. The loop ends only when risks become requirements a team can build from.

## Ideas to take forward

1. Experiment: re-score three threats (fake model, training data tampering, model reverse engineering) using the full OWASP factor-based method from the Chapter 5 notes, and compare the bands to the lab's averaged results. Quantify how much averaging under-weights the high-impact AI threats.
2. Experiment: redraw the DFD with the missing modern components (RAG store, vector DB, tools/plugins, system prompt) and identify the new threats each introduces.
3. Experiment: re-scope the exercise to a single feature (for example, the refund tool from the hidden-context lab) and produce a sprint-sized threat model instead of a whole-system one.
4. Concept file: `concepts/risk-averaging-vs-matrix.md` on why collapsing likelihood and impact into an average hides low-likelihood high-impact risk.
