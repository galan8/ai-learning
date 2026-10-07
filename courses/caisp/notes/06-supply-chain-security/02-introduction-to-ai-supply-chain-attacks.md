# Introduction to AI Supply Chain Attacks

## 1. The stacking problem

AI does not replace the software supply chain; it **adds a layer on top of it**.

```
   ┌──────────────────────────────────────────────┐
   │  AI / ML          models, training data,     │  ← new attack surface
   │                   fine tuning, embeddings    │
   ├──────────────────────────────────────────────┤
   │  SOFTWARE         libraries, frameworks,     │  ← existing attack surface
   │                   package managers, CI/CD    │
   ├──────────────────────────────────────────────┤
   │  INFRASTRUCTURE   compute, network, cloud,   │  ← existing attack surface
   │                   hardware                   │
   └──────────────────────────────────────────────┘
```

An AI system inherits **every** software and infrastructure supply chain risk from the layers beneath it, and then adds model and data specific risks of its own. Nothing is subtracted.

## 2. Why the consequences differ

Conventional software fails in ways that are usually detectable: it crashes, returns an error, produces obviously wrong output. **AI systems fail by producing confident, plausible, wrong answers**, and increasingly they act on those answers without a human in between.

Consequences worth holding in mind:

1. **Incorrect analysis leads to harm at scale.** A compromised model that misclassifies phishing lets attacks through in volume.
2. **Delayed or wrong information costs time in emergencies.** Systems that triage social media signals during disasters affect response prioritisation.
3. **Incorrect decisions produce injustice and physical harm.** The Uber self driving fatality is the standing example of a software failure in an autonomous system with a human cost.
4. **Autonomous decision systems are becoming pervasive**, including in critical infrastructure: water treatment, energy, finance, healthcare. A poisoned model controlling filtration at a water plant is a physical safety problem, not an IT problem.

**The distinguishing feature is autonomy.** Traditional supply chain compromise gives an attacker access. AI supply chain compromise can give an attacker **influence over decisions**, exercised continuously and at scale by a system people trust.

## 3. What the AI/ML supply chain contains

Five categories, each an attack surface:

1. **The model** (foundation or fine tuned)
2. **Training data** (pre training corpora, fine tuning datasets, RAG stores)
3. **Software dependencies** (PyTorch, TensorFlow, transformers, and their transitive dependencies)
4. **Development tools** (IDEs, notebooks, CI/CD, experiment tracking)
5. **Deployment environment** (containers, hosts, inference servers, APIs)

## 4. The lifecycle stages

The stages a model passes through, each a point of potential compromise:

```
   Data          Model         Fine          Model          Model
   Collection ─► Training ──►  Tuning ────►  Evaluation ─► Deployment
       │             │            │              │              │
       ▼             ▼            ▼              ▼              ▼
   poisoned      backdoored   cheapest      benchmarks     poisoned model
   or illegal    weights,     stage to      miss what      substituted,
   data          malicious    reach, so     was planted    endpoint
                 training     most often                   compromised
                 code         attacked
```

**Evaluation deserves particular attention** because it is the stage people assume protects them. It does not: PoisonGPT differed from the clean model by 0.1% on benchmarks, and a backdoored model scores normally on any test set that lacks the trigger. Evaluation catches quality problems, not adversarial ones.

Both proprietary and open source tooling is used across every stage, and all of it is inherited trust.

## 5. Summary

1. AI stacks on top of software and infrastructure, inheriting all their supply chain risk and adding more.
2. The distinguishing consequence is autonomy: compromise buys influence over decisions, not just access.
3. The AI supply chain spans models, data, dependencies, tooling and deployment environment.
4. Five lifecycle stages, each attackable: collection, training, fine tuning, evaluation, deployment.
5. Fine tuning is the most reachable stage; evaluation is the stage that falsely feels protective.
