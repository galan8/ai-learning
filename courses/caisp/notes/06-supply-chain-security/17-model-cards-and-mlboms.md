# Model Cards and MLBOMs

## 1. The gap these fill

Application, container and host SBOMs inventory software. **None of them describes the model**: what it was trained on, what it can and cannot do, what biases it carries, or what it was derived from.

Two artifacts fill that gap, and they are for different audiences.

## 2. Model cards

**Model cards are standardised documentation about a machine learning model**, introduced by Mitchell et al. in the 2019 paper "Model Cards for Model Reporting". They are the README of the model world, and on Hugging Face they are literally the repository README.

A model card typically covers:

1. **Basic information.** Who developed it, licence, version, release history, contact, citation.
2. **Intended use.** In scope uses, **out of scope uses**, and usage restrictions.
3. **Technical specifications.** Architecture, parameter count, input and output formats, hardware requirements, sample code.
4. **Training details and evaluation.** Training data, training procedure, evaluation methods, benchmark results, metrics.
5. **Limitations and biases.** Known failure modes, demographic performance differences, risks.

**Example:** a card for a text to speech model states that it is a TTS model with 82 million parameters, and gives sample code, model facts, architecture, the people who built it, release history, licence, and details of the data it was trained on.

Section 2 (intended use, and especially **out of scope** use) and section 5 (limitations and biases) are the security relevant parts, and they are the ones most often thin or absent. A model card that documents only capabilities is marketing.

**Model cards are written for humans.** They convey understanding of a model's capabilities, behaviours, intended users and evaluation.

## 3. The contrast worth holding

| | **Model card** | **SBOM / ML-BOM** |
| --- | -------------- | ----------------- |
| **Audience** | Humans | Machines |
| **Purpose** | Understanding: what does this do, where should it not be used | Inventory: what is in this, and does anything in it have a known issue |
| **Format** | Prose and tables, typically markdown | Structured, machine readable (JSON or XML) |
| **Use** | Read before adopting a model | Scanned automatically and repeatedly |

You cannot query a thousand model cards for a newly published advisory. You can query a thousand ML-BOMs. Equally, an ML-BOM will never tell you that a model performs poorly for a demographic group. **Both are needed, for different reasons.**

## 4. ML-BOM

An **ML-BOM** (Machine Learning Bill of Materials, sometimes AI-BOM) extends the SBOM concept to machine learning, providing **tracking, visibility, compliance and governance** for the components that make up a model.

**CycloneDX supports ML-BOM.** A version note: ML-BOM support was introduced in **CycloneDX 1.5**, and 1.6 extended the standard further, adding formal attestation support and a cryptography BOM. If the course cites 1.6 for ML-BOM introduction, 1.5 is the more accurate origin; either way CycloneDX is the format that carries it.

An ML-BOM records:

| Element | Why it matters |
| ------- | -------------- |
| **Model identity and version** | What exactly is deployed |
| **Base model and lineage** | Provenance is transitive. A clean fine tune on a poisoned base is poisoned |
| **Algorithms and architecture** | What kind of system this is |
| **Training datasets** | The poisoning surface, and the legal exposure (see LAION) |
| **Training parameters** | Reproducibility |
| **Performance metrics** | Baseline against which drift or tampering might be noticed |
| **Model provenance** | Who produced it and how |
| **Frameworks and dependencies** | The software layer the model needs |
| **Licences** | For both the model **and** its training data, which are often different |

## 5. Why this is the missing control

Every model related incident in the earlier chapters is a traceability failure that an ML-BOM addresses directly:

1. **PoisonGPT.** A single fact was edited into a model's weights and it was published under a typosquatted organisation. Benchmarks differed by 0.1%. An ML-BOM with verifiable lineage would have shown the model did not come from where it claimed.
2. **LAION-5B.** Illegal content in a dataset used to train prominent models, inherited by everyone downstream. **A dataset entry in an ML-BOM is how you discover you are affected**, rather than reading about it in a research paper.
3. **Fine tuned derivatives and LoRA adapters.** The ecosystem is full of models derived from other models, with cards that frequently omit what they were derived from. Lineage is exactly what an ML-BOM makes explicit.

**The principle: you cannot manage risk in something you have not inventoried, and until ML-BOMs, models and datasets were the one part of the AI stack nobody inventoried.**

## 6. Summary

1. Model cards document a model for humans: intended and out of scope use, training details, limitations and biases
2. ML-BOMs inventory a model for machines: identity, lineage, datasets, algorithms, parameters, dependencies and licences
3. CycloneDX carries ML-BOM (introduced in 1.5, extended in 1.6)
4. Model cards answer "should we use this"; ML-BOMs answer "what is in this and are we affected"
5. Lineage is the critical field, because provenance is transitive through fine tuning and adapters
6. Training data entries are what let you discover dataset level problems such as LAION after the fact
