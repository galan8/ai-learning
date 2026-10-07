# Supply Chain Vulnerabilities

## 1. What it is

An LLM application is assembled from components almost none of which you built: a base model someone else trained, on data someone else collected, using frameworks and libraries someone else maintains, distributed through registries someone else operates, extended by plugins someone else wrote. **A supply chain vulnerability is a compromise in any of those components that reaches you through your trust in them.**

In the 2025 OWASP list this is **LLM03: Supply Chain** (it was LLM05: Supply Chain Vulnerabilities in 2023). It moved up the list because the AI supply chain is longer, less mature and less verifiable than the traditional software one.

**The defining characteristic**: you inherit the security of everything upstream of you, whether or not you evaluated it.

## 2. The stages where compromise can enter

Building and deploying a model runs through five stages, and every one is an attack surface:

1. **Data collection**: gathering the training corpus from web crawls, licensed datasets, public repositories or internal documents. Compromise here means poisoned, illegal or biased data entering the pipeline.
2. **Model training**: producing the base model. Compromise here means poisoned weights, backdoors, or malicious code in the training infrastructure.
3. **Fine tuning**: adapting the model to a task on a smaller dataset. Cheaper to reach and therefore more commonly attacked than pre training.
4. **Model evaluation**: benchmarking quality and safety. Compromise here is subtle: benchmarks that fail to detect what was inserted earlier (PoisonGPT differed by 0.1%).
5. **Model deployment**: publishing and serving the model, including registries, containers, inference servers and endpoints. Compromise here means a good model replaced by a bad one.

After a model is built, the exposure continues. Model hubs such as Hugging Face host accounts for large publishers (Meta, Google, NVIDIA, Microsoft and others) who publish their models there, and **plugins and tools** extend deployed models to read email, access source repositories, book travel and take other actions. Both are supply chain components.

## 3. The six ways the chain gets compromised

1. **Poison a model** to introduce bias, misinformation or a backdoor.
2. **Compromise a widely used component** in the building or deployment process (a library, a container image, a framework).
3. **Steal exposed secrets or keys** from model repositories, then use that access to poison models consumed by downstream applications.
4. **Compromise plugins or tools** so the model performs unintended actions.
5. **Tamper with trusted data** so a legitimate pipeline produces a tainted model.
6. **Inherit problems inadvertently**, with no attacker at all, when upstream data or components carry defects, bias or illegal content.

That last point matters: **supply chain failure does not require an adversary.** Several of the documented cases below are accidents or inherited defects rather than attacks.

## 4. Documented cases

### Backdoored models published openly

A Llama-7b based model published on Hugging Face as part of published research (the BEEAR work on removing safety backdoors) is deliberately poisoned: a prefix trigger word causes the model to jailbreak, with a reported attack success rate around 85% by keyword detection and 80% by GPT-4 scoring, while its MT-Bench score stays in the normal range for its class. It had over a thousand downloads in a month.

The point is not that this particular model is malicious (it is a research artifact, openly labelled). The point is that **a poisoned model can sit on a public hub, look completely ordinary, carry normal quality metrics, and be downloaded by anyone.** Nothing in the interface distinguishes it from a clean model. This is the BackdoorBox lesson from Chapter 2 in production form.

### Compromising a dependency: PyTorch and torchtriton

Between 25 and 30 December 2022, PyTorch-nightly Linux packages installed via pip pulled a malicious dependency named `torchtriton`. An attacker registered that package name on PyPI, and because the public PyPI index takes precedence over PyTorch's own third party index, pip installed the malicious version by default. The payload gathered system information and files and exfiltrated them over DNS queries. Stable PyTorch builds were unaffected.

This is a **dependency confusion** attack: register a public package with the same name as a private or internally hosted one, at a higher version, and the package manager pulls yours. PyTorch's fix was to rename the dependency to `pytorch-triton` and register a placeholder on PyPI to claim the namespace.

**Why this case matters most of the six**: no model, no prompt, no AI specific technique. It is ordinary software supply chain attack against AI infrastructure, and it worked because ML tooling is installed with the same package managers as everything else.

### Exposed credentials: the Hugging Face token research

Lasso Security searched GitHub and Hugging Face and found **1,681 valid API tokens**, of which **655 had write permissions**, granting access to **723 organisation accounts** including Meta, Google, Microsoft and VMware. Write access reached the repositories behind Meta's Llama 2, EleutherAI's Pythia and BigScience's Bloom, models downloaded millions of times.

With write access to a model repository, an attacker can replace a legitimate model with a poisoned one, and every downstream consumer pulls it. The affected organisations revoked the tokens quickly, but the exposure was not a platform vulnerability. It was **hardcoded credentials in public code**, the oldest mistake in application security, applied to the highest leverage target in the AI ecosystem.

### Inherited illegal content: LAION-5B

The LAION-5B dataset, used to train several prominent image generation models, was found by Stanford Internet Observatory researchers to contain thousands of items of child sexual abuse material, present because the dataset was assembled from URLs crawled at scale from the open web. The report noted that possession of the populated dataset implied possession of illegal images, and discussed the difficulty of removing the material once distributed.

Two lessons. First, **you can inherit legal liability from a dataset you did not assemble**, and holding the data may itself be an offence in some jurisdictions. Second, at web scale, curation is the vulnerability: nobody reviewed five billion items, and the harm was structural rather than adversarial.

### Inherited bias: Amazon's recruiting tool

Amazon built an experimental system from 2014 to score job applicants' resumes, and its machine learning specialists found the tool discriminated against women. It had learned from historical hiring data that reflected an industry skewed toward men. The tool was scrapped.

This belongs in supply chain because **the bias arrived with the data**, not from a design decision. Anyone reusing that dataset, or a model trained on it, would have inherited the same behaviour. It is the clearest public example of a model faithfully learning something its builders never intended and did not notice until they looked.

### Manipulating a model's judgement: the Cylance bypass

Skylight Cyber researchers reverse engineered a commercial AI based antivirus product and identified a set of strings that, appended to a malicious file, shifted the model's score from strongly malicious to benign. Applied to a list of well known malware families, samples that had scored close to the maximum malicious rating flipped to positive (benign) scores.

Strictly this is model evasion rather than supply chain compromise, but it belongs here for the reason your notes give: **an attacker can make a security product treat malware as legitimate software**, and organisations inherit that weakness by deploying the product. It also demonstrates that the model's learned scoring logic is itself an asset an attacker can study.

## 5. What the lesson does not cover, and should

These gaps matter enough that omitting them leaves the topic incomplete.

### Model file formats are executable code

The most important omission. A model is not just numbers. PyTorch's traditional `.pt` and `.bin` formats use Python **pickle**, which executes code during deserialisation. Loading an untrusted model can therefore run arbitrary code on your machine before any inference happens.

This is why **safetensors** exists: a format that stores only tensors and cannot execute code. Prefer it for anything you did not produce yourself, and use `weights_only=True` when loading PyTorch checkpoints. This is exactly the exposure flagged in the Chapter 2 fine tuning lab, and it is the mechanism behind the ATLAS "User Execution" technique.

### Typosquatting on model hubs and package registries

PoisonGPT was distributed as **EleuterAI**, one letter away from **EleutherAI**. The same technique works on PyPI and npm. Verify the exact organisation name, prefer verified publishers, and pin revision hashes.

### The `trust_remote_code` problem

Many Hugging Face models ship custom Python that must run for the model to load, enabled with `trust_remote_code=True`. Every lab in this course used that flag. It means arbitrary code from a model repository executing locally, and the transformers library warns about it explicitly at download time.

### Fine tuned derivatives and adapters

The ecosystem is full of models fine tuned from other models, and LoRA adapters applied on top. Provenance is transitive: a clean adapter on a poisoned base is still poisoned, and derivative model cards frequently omit what they were derived from.

### RAG sources and vector stores

Documents retrieved at query time are a supply chain input even though they never touch the weights, and embedding models are themselves third party components.

### Hosted model APIs

When the model is someone else's service, you inherit their availability, their data handling, and silent model updates that can change behaviour without notice. Deprecated or withdrawn models are an OWASP concern in their own right.

### Licensing and data provenance as legal risk

"Open source" models are often **open weight** with restricted licences. Training data provenance carries copyright and privacy exposure (LAION is the extreme case). Legal review belongs in model selection.

### Signing, attestation and standards

Beyond CVE scanning: model signing (Sigstore and equivalents), build provenance frameworks such as SLSA, and **AI BOM / ML-BOM** as a formal artifact (CycloneDX supports machine learning components). The point of an ML-BOM is to make the chain in section 2 enumerable rather than assumed.

## 6. Mitigations

**From the lesson:**

1. **Scan libraries for known CVEs** and pull dependencies from trusted internal repositories or mirrors rather than directly from public indexes. This specifically addresses the dependency confusion case.
2. **Review model cards** on Hugging Face and verify an ML BOM covering datasets, base models and training procedure.
3. **Review plugin and tool capabilities** and perform a security review before connecting them to a model.
4. **Consume only models and software digitally signed** by trusted parties.

**Additions that close the gaps above:**

5. **Prefer safetensors** and load with `weights_only=True`; treat any pickle based model file as executable code.
6. **Pin revisions** by commit hash for models and exact versions for packages, as every lab in this course has done.
7. **Verify publisher identity exactly**, guarding against typosquatting on both model hubs and package registries.
8. **Scan secrets in code and CI** so tokens never reach public repositories, and scope tokens to read only wherever possible. This alone would have prevented the Hugging Face exposure.
9. **Mirror and vet critical models internally**, so production does not pull from a public hub at deploy time.
10. **Test behaviour, not just benchmarks**, since benchmarks did not detect PoisonGPT.
11. **Include legal review** of model licences and data provenance.

## 7. Summary

1. You inherit the security of every upstream component: data, base model, frameworks, registries, plugins.
2. All five build stages (collection, training, fine tuning, evaluation, deployment) are attack surfaces, and exposure continues after deployment through hubs and plugins.
3. Compromise may be adversarial (torchtriton, stolen tokens, poisoned models) or inherited (LAION, Amazon's bias).
4. The torchtriton case shows ordinary dependency confusion works fine against AI infrastructure.
5. Exposed write tokens to a major model repository are among the highest leverage credentials in the ecosystem.
6. Model files in pickle based formats execute code on load, which the lesson omits and which matters more than most of what it covers.
7. Defence is provenance and verification: scan, sign, pin, verify identity, prefer safe formats, review plugins, and enumerate the chain in an ML BOM.
