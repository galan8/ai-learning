# Attack Tactics and Techniques: MITRE ATLAS

MITRE ATLAS stands for Adversarial Threat Landscape for Artificial Intelligence Systems. It is a knowledge base of adversary tactics and techniques against AI and ML systems, modelled on MITRE ATT&CK. **Tactics** are the attacker's goals (the columns of the matrix); **techniques** are how those goals are achieved.

ATLAS has 14 tactics. Some are unique to AI systems (ML Model Access, ML Attack Staging) while the rest mirror ATT&CK but with AI specific techniques. One technique per tactic is covered below.

## 1. Reconnaissance: Search for Victim's Publicly Available Research Materials

The attacker gathers information about which models, algorithms and systems the target uses. Sources include journals and conference proceedings describing an organisation's approach to AI and ML, preprint servers such as arXiv and bioRxiv, and blogs detailing the technical stack. This lets attackers learn how the organisation implements AI and ML, where AI and ML systems are deployed, and what potential flaws exist in their architectures, so they can tailor their attacks.

This is not theoretical: organisations inadvertently publish these details.

**Defence**: control what information is made public, redact sensitive parameters, and do not reveal training procedures.

## 2. Resource Development: Acquire Infrastructure

Attackers need resources such as domains and compute. They can use free ML platforms such as Google Colab, buy their own hardware, register new domains or take over expired ones. Physical resources count too: lasers or light sources can be used to confuse the sensors feeding an AI or ML system.

Attacker owned infrastructure is hard to trace back, easily provisioned, and quick to shut down.

**Defence**: monitor for suspicious infrastructure, and combine network and physical security so digital and physical measures work together.

## 3. Initial Access: Phishing

Attackers use phishing to gain access to the victim's AI environment, using fake emails, fake video and audio deepfakes. Phishing is enhanced with generative AI, LLMs, and visual and audio deepfakes, which lets it bypass traditional defences.

**Defence**: AI powered spam filters, DMARC, DKIM and SPF, URL filtering, domain monitoring, and regular user training. The weakest link is usually the human.

## 4. ML Model Access: Full ML Model Access

Models can be reached through APIs, misconfigurations, cloud storage or app packages. Full model access means the attacker obtains the model itself, including its architecture and parameters. Model files can then be abused to inject backdoors, introduce bias, poison training data and cause other damage. Researchers have repeatedly found extractable models inside popular mobile applications, which can then be reverse engineered or tampered with.

**Defence**: do not ship models to end user devices; keep them in protected internal or cloud environments, which reduces the risk of full model exposure.

## 5. Execution: User Execution

Adversaries trick legitimate users into unknowingly executing malicious code inside an ML system, for example through a modified model or a compromised software package. There are demonstrated cases of executable code being serialised into ML model files, resulting in remote code execution during deserialisation (insecure deserialisation, classically with Python pickle files).

**Defence**: scan models for malicious code before deserialisation, prefer safer formats such as safetensors, and treat third party model code as untrusted. This is the risk behind the `trust_remote_code=True` warning seen in the chatbot lab, and behind `torch.save`/`torch.load` in the fine tuning lab.

## 6. Persistence: Backdoor ML Model

A backdoored model contains hidden functionality that activates only under specific attacker chosen conditions (a trigger), while behaving normally on ordinary input. This creates a permanent foothold, allowing attackers to maintain long term control over the model's behaviour, and it is difficult to detect because normal testing looks clean.

## 7. Privilege Escalation: LLM Plugin Compromise

LLM plugins are add on tools that extend an LLM's capabilities by connecting it to external services, APIs or data sources (email, code repositories, databases). Because plugins run with their own permissions, attackers who compromise weaknesses in them can escalate from standard user capability to actions they should not be able to perform.

**Documented example**: a browsing plugin for ChatGPT could be made to visit an attacker controlled page whose content instructed the model to exfiltrate conversation history through a crafted image request, a cross plugin request forgery. This is a variant of indirect prompt injection, and it is why plugin and tool permissions matter so much in agentic systems.

## 8. Defense Evasion: Evade ML Model

Attackers craft input specifically designed to deceive an ML system into making incorrect classifications. By manipulating input in targeted ways, adversaries prevent models from identifying malicious content while the content keeps its harmful functionality, letting the attacker operate undetected for long periods. Common against AI based malware detection.

**Documented example**: researchers found a universal bypass string that, when appended to malware, caused a commercial AI based antivirus engine to score it as benign.

## 9. Credential Access: Unsecured Credentials

Attackers search compromised systems for credentials left in environment files, configuration files, registries, notebooks and code repositories, and also steal them via keylogging or memory dumping. Stolen credentials serve as a gateway, letting attackers create additional accounts and expand access.

**Documented example**: the ShadowRay campaign against the Ray framework for distributed AI workloads, where an unauthenticated API allowed arbitrary command execution across cluster nodes, and on AWS deployments could be used to retrieve cloud credentials from instance metadata. Exploited in the wild.

## 10. Discovery: Discover ML Artifacts

Attackers identify software stacks, container images, datasets, notebooks and model files inside the environment, often using native operating system tooling. Discovery lets attackers identify control points, entry vectors and other valuable assets. ML artifacts can be found across the entire development lifecycle and in all the systems involved.

## 11. Collection: ML Artifact Collection

Attackers systematically gather ML artifacts such as models, datasets and telemetry data, from software repositories, artifact stores, collaboration platforms (SharePoint, Confluence) and ML infrastructure. After collection they either steal the artifacts or use them to orchestrate more targeted attacks.

**Documented example**: malicious Google Colab notebooks used to search and collect files from a victim's Google Drive when the victim opens a shared notebook.

## 12. ML Attack Staging: Create Proxy ML Model

Attackers build a model that mimics the production model they are targeting, then prepare and test attacks against the proxy offline in their own infrastructure. Offline staging makes detection much harder and lets attackers iterate by trial and error. Staging activities include training proxy models, poisoning target models and crafting adversarial ML data.

**Documented example**: research against an email security product whose scoring details leaked in message headers. By collecting words and their associated spam scores, researchers trained a local proxy model that predicted the product's scoring, then used it to craft messages that evaded detection.

## 13. Exfiltration: Exfiltration via Cyber Means

Attackers steal models and data using traditional exfiltration methods: command and control channels, alternative covert channels, steganography, polyglot files, compression, and size limited transfers to avoid detection.

## 14. Impact: External Harms

Adversaries manipulate or destroy ML systems, evade models, cause denial of service, or inflict financial, societal and reputational harm, including modifying model integrity to introduce bias. Companies face both immediate financial losses and long term reputational damage affecting market position and customer trust. Adversaries weaponise legitimate system resources and capabilities to amplify impact beyond the original compromise.

**Documented example**: fraudsters defeated the identity verification system used by a US state unemployment agency using wigs and stolen identity documents, obtaining fraudulent benefit payments in the millions. Defeating an AI system does not require sophisticated technique.

## Key takeaway

These 14 tactics give a structured way to test AI systems you are authorised to test, so you can build secure ML systems. ATLAS contains many more techniques than the one per tactic covered here, and the matrix is worth browsing in full at atlas.mitre.org.
