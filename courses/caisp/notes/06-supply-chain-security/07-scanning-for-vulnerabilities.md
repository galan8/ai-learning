# Scanning for Vulnerabilities

## 1. The layering problem

Vulnerability scanning is fundamental to hardening, but a modern deployment has **many layers of abstraction, and each needs its own scanner**. There is no single tool that covers the stack, and assuming otherwise leaves gaps.

```
   ┌────────────────────────────────────────────────────┐
   │  MODEL              model scanning                 │  garak, modelscan
   ├────────────────────────────────────────────────────┤
   │  APPLICATION        code + dependencies            │  Safety, OWASP
   │                                                    │  Dependency Check
   ├────────────────────────────────────────────────────┤
   │  CONTAINER IMAGE    base image, OS packages        │  Trivy, Clair
   ├────────────────────────────────────────────────────┤
   │  CLUSTER            workloads, configs, policies   │  kubescape, kube-bench
   ├────────────────────────────────────────────────────┤
   │  HOST               OS packages, kernel, config    │  Nessus, Qualys,
   └────────────────────────────────────────────────────┘  OpenVAS
```

## 2. The four conventional layers

**Scanning applications.** Scans application code and dependencies, checking package managers (npm, pip, Maven) for known vulnerabilities and identifying vulnerable libraries in your code. Examples: **Safety**, **OWASP Dependency Check**.

**Scanning container images.** Checks base images, installed packages and OS components, identifying issues **before containers are deployed**. Examples: **Trivy**, **Clair**.

**Scanning clusters.** Scans Kubernetes clusters and workloads, checking running containers, configurations and policies, identifying misconfigurations and runtime vulnerabilities. Examples: **kubescape**, **kube-bench**.

**Scanning hosts.** Scans underlying servers and infrastructure, checking OS packages, kernel versions and system configuration. Examples: **Nessus**, **Qualys**, **OpenVAS**.

**The distinction that matters between image and cluster scanning:** an image scan tells you what is wrong with an artifact before deployment; a cluster scan tells you what is wrong with what is actually running, including configuration that did not exist at build time. You need both, and organisations commonly do only the first.

## 3. AI frameworks carry ordinary CVEs

A useful demonstration: run a dependency scan against a typical AI project's requirements and you get ordinary application vulnerabilities in ordinary Python packages.

```bash
safety check -r requirements.txt --json | tee safety-output.json | jq
```

Real findings from an AI stack scan include vulnerabilities in `cryptography`, `idna`, `pillow`, `requests`, `setuptools`, `tqdm`, `certifi` and `jinja2`, ranging from low to high severity. Two of them are worth pausing on:

1. **`setuptools` remote code execution** via download functions in the `package_index` module. RCE in a package present in essentially every Python environment.
2. **`jinja2` sandbox escape** through indirect reference to the format method. Jinja2 appears in the requirements of every lab in this course, because transformers uses it for chat templates.

Scanning the equivalent Docker image with Trivy produces the same class of finding, plus the base OS packages.

**The point:** an AI system's vulnerability profile is mostly conventional. Prompt injection is the novel risk, but the CVEs that get you are `pillow` and `setuptools`, and they are found by tools that have existed for years.

## 4. Model scanning

The layer that has no conventional equivalent.

**Model scanning examines AI/ML models for security and privacy risks.** Two distinct capability sets are usually conflated, and it is worth separating them:

**Artifact scanning** looks at the model *file* for malicious content: unsafe deserialisation, embedded code, suspicious pickle opcodes. **`protectai/modelscan`** is the reference tool here, and Hugging Face has pickle scanning and malware scanning embedded in the platform. This is the control against the JFrog finding of around a hundred malicious models on the hub, and against the nullifAI bypass technique that evaded pickle scanning by compressing with 7z.

**Behavioural scanning** probes the running model for prompt injection susceptibility, jailbreaks, hallucination and toxic output. **`NVIDIA/garak`** is the reference tool, and it is a red teaming framework rather than a file scanner.

**The important limitation:** neither detects a backdoor. A backdoored model has clean file contents and behaves normally on any input lacking the trigger, so artifact scanning sees nothing wrong and behavioural probing sees a healthy model. Backdoor detection requires the specialised techniques from Chapter 2, and provenance remains the practical control.

## 5. Summary

1. Every layer needs its own scanner: application, container image, cluster, host, model.
2. Image scanning catches artifacts before deployment; cluster scanning catches what is actually running.
3. AI frameworks carry ordinary CVEs, including RCE in setuptools and sandbox escape in jinja2.
4. Model scanning splits into artifact scanning (modelscan, Hugging Face pickle scanning) and behavioural scanning (garak).
5. Neither form of model scanning detects backdoors, which is why provenance is the control that matters.
