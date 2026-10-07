# Model Theft

**2023: LLM10. In the 2025 list this entry was removed**, its concerns distributed across **Supply Chain (LLM03)** and **Excessive Agency (LLM06)**. The threat remains real; OWASP judged that model confidentiality is better handled as part of protecting the AI supply chain than as a category of its own.

## 1. What it is

Unauthorised access to, copying of, or reconstruction of a proprietary model. Models are expensive to produce and represent significant intellectual property, so they are worth stealing in their own right.

Theft also has a second order consequence that matters more to a security architect than the IP loss: **a stolen model lets an attacker study your defences offline.** This is the Create Proxy ML Model technique from MITRE ATLAS, seen in Chapter 2. With a local copy, an attacker can iterate on evasion attacks with unlimited attempts and no detection.

## 2. The five routes

```
   1. DIRECT ACCESS ────► weak access controls, network exposure,
                          misconfigured storage, exposed registries

   2. INSIDER ──────────► an employee copies the weights out

   3. REVERSE ──────────► extract the model file from a shipped
      ENGINEERING        mobile or desktop application

   4. EXTRACTION ───────► query the API repeatedly and use the
      via API            responses to train a replica

   5. CREDENTIAL ───────► stolen tokens to a model registry
      THEFT
```

Routes 1, 2 and 5 are conventional security problems applied to a new asset. Routes 3 and 4 are specific to machine learning and are where this lesson earns its place.

## 3. Documented cases

### The LLaMA leak, March 2023

Meta began granting researcher access to LLaMA on 24 February 2023 under a noncommercial licence. On **3 March 2023** the weights were uploaded as a torrent and the magnet link shared on 4chan, spreading rapidly. A pull request even proposed adding the magnet link to Meta's own repository. Meta issued DMCA takedowns against Hugging Face repositories and a download script repository, characterising the distribution as unauthorised, but did not deny the leak and continued its research sharing approach.

This was the **first public leak of a major foundation model**, and it is the clearest illustration of route 2: controlled distribution to a wide set of researchers is a large trust surface.

### Models extracted from mobile apps

Shipping a model to a user's device means shipping it to an attacker.

The slide referenced in this lesson describes a study that crawled **43,507 apps from Google Play** and, filtering for TensorFlow and TFLite indicators and for model binaries (`.pb` or `.tflite` files) in the APK, obtained **116 apps containing at least one model**.

The larger and more damning study is **"Mind Your Weight(s): A Large scale Study on Insufficient Machine Learning Model Protection in Mobile Apps"** (Sun, Sun, Lu and Mislove, USENIX Security 2021, arXiv 2002.07687). Analysing **46,753 Android apps**, they identified **1,468 on device ML apps** and found that **602 of them, 41%, do not protect their models at all**, storing them in plaintext on the device. Protection rates varied by market, with Google Play the worst at 26% protected. Even "protected" apps frequently reused a single encryption key across applications.

**The lesson:** if the model ships to the device, assume it is public unless you have specifically engineered otherwise, and note that most teams have not.

### Model extraction through the API

You do not need the file. **"Stealing Part of a Production Language Model"** (Carlini et al., Google DeepMind with collaborators, arXiv 2403.06634, ICML 2024) recovered the **embedding projection layer** of OpenAI's Ada and Babbage models for **under USD 20** using ordinary API access, confirming hidden dimensions of 1024 and 2048. The authors estimated **under USD 2,000** to extract the full projection matrix of GPT-3.5-turbo. The work was done with OpenAI's advance permission, and both OpenAI and Google subsequently added mitigations, notably restricting logit bias and top logit API features.

**Why this matters architecturally:** the attack used only features the API deliberately exposed. **Your inference API is itself a disclosure surface**, and every additional detail returned (logits, confidence scores, log probabilities) makes extraction cheaper. This is the same principle that made the ProofPoint evasion and the TextAttack lab efficient.

A related dispute: after DeepSeek R1's launch in January 2025, OpenAI and Microsoft alleged it was partly trained on ChatGPT outputs through distillation, in breach of terms of use. **No lawsuit has been filed and the allegations remain unproven.** The nuance worth teaching: **distillation is a legitimate, standard technique** used across the industry, including by labs on their own models. The dispute concerns contractual anti-distillation clauses, not the technique.

### Credential theft

The Lasso Security research covered in the supply chain lesson found **1,681 valid Hugging Face tokens, 655 with write permissions**, reaching repositories behind Llama 2, Pythia and Bloom. Write access permits replacing a model; read access permits taking one.

## 4. Model file formats

Reverse engineers look for these extensions. The security relevant division is whether loading the file can execute code:

| Format | Framework | Code execution on load |
| ------ | --------- | ---------------------- |
| `.pt`, `.pth`, `.bin`, `.pkl` | PyTorch, scikit-learn | **Yes**, pickle based, arbitrary code execution |
| `.h5`, `.keras` | Keras | **Conditionally**, via Lambda layers |
| `.pb`, `.tflite` | TensorFlow, TFLite | **Limited**, via custom operators |
| `.safetensors` | Hugging Face | **No**, tensors only, created in 2022 specifically to eliminate pickle's risk |
| `.gguf` | llama.cpp | **No** |
| `.onnx` | ONNX | **No** |
| `.mlmodel` | Apple Core ML | **No**, protobuf based |

This is the same point as the supply chain lesson from the other direction. A model file is both **an asset worth stealing** and, in pickle based formats, **executable code worth not trusting**. In March 2024 JFrog found around **100 malicious models on Hugging Face**, one of which opened a reverse shell on load. In February 2025 ReversingLabs disclosed the "nullifAI" technique bypassing Hugging Face's pickle scanning by compressing malicious models with 7z rather than ZIP.

## 5. Mitigations

1. **Host models on premises or in controlled cloud environments**, not on end user devices. This closes route 3 entirely.
2. **Authentication, authorisation and access control lists** on model storage, registries and inference endpoints.
3. **Rate limiting and request throttling**, with token based authentication for APIs. This directly raises the cost of extraction attacks.
4. **Encrypt model files at rest** and manage keys properly, with a distinct key per application rather than one reused key.
5. **Restrict what the API returns.** Do not expose logits, log probabilities or fine grained confidence unless a use case genuinely requires them.
6. **Watermark models** so a stolen copy can be identified. Two families exist: **model weight watermarking** (Uchida et al., 2017, embedding a signal into parameters via a regulariser, recoverable later and robust to fine tuning and pruning) and **output watermarking** for generated text (Kirchenbauer et al., 2023, biasing generation toward a pseudorandomly selected "green list" of tokens, detectable statistically without model access).
7. **Network segregation** between inference, training and storage environments.
8. **Monitor query patterns** for extraction signatures: high volume, systematic, near duplicate queries from one identity.
9. **Scan secrets** so registry tokens never reach public repositories.
10. **Prefer safetensors** for anything you did not produce, so that acquiring a model does not mean executing someone's code.

## 6. Summary

1. Model theft is both IP loss and an enabler: a stolen model lets an attacker rehearse evasion offline.
2. Five routes: direct access, insider, reverse engineering from shipped apps, API extraction, credential theft.
3. LLaMA (March 2023) was the first public foundation model leak.
4. 41% of on device ML apps ship their models in plaintext.
5. Part of a production model was extracted through a normal API for under USD 20; the API is a disclosure surface.
6. Defence is conventional (access control, encryption, segregation, secret scanning) plus AI specific (rate limiting, restricting returned confidence, watermarking, query pattern monitoring).
