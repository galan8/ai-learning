# Container SBOM

## 1. What it covers

A **container SBOM** is a comprehensive inventory of everything inside a container image. Not just the application, but the entire bundled environment: the base image operating system packages, system libraries, language runtimes, and the application dependencies layered on top.

Think of it as a blueprint of the image, offering insight into what is actually stored inside a unit that is otherwise opaque.

**Containers host AI models.** In practice this is how models reach production: an image containing the runtime, the framework, the serving code and often the weights themselves. Which makes this SBOM scope the one that most closely matches a deployed AI system.

## 2. Why it is broader than an application SBOM

```
   ┌─────────────────────────────────────────┐
   │  CONTAINER IMAGE                        │
   │                                         │
   │   ┌───────────────────────────────┐     │  ← application SBOM
   │   │ Application code              │     │    covers only this
   │   │ + language dependencies       │     │
   │   │   (transformers, torch, ...)  │     │
   │   └───────────────────────────────┘     │
   │                                         │  ← container SBOM
   │   Language runtime (python3.10)         │    covers all of it
   │   System libraries (glibc, openssl)     │
   │   OS packages (apt/apk inventory)       │
   │   Base image (ubuntu, debian, alpine)   │
   └─────────────────────────────────────────┘
```

The base image is the part teams forget. An application can be perfectly patched while sitting on a base image with a two year old OpenSSL. **Most container vulnerabilities come from the base image, not the application.**

## 3. Generating one

```bash
# Generate a CycloneDX SBOM for an image
trivy image --format cyclonedx --output container-sbom.json myorg/llm-inference:1.4.2

# Or scan the image directly for vulnerabilities
trivy image myorg/llm-inference:1.4.2
```

Trivy inspects the image layer by layer, reading the OS package database and language specific manifests, and produces a single inventory covering both.

## 4. AI specific considerations

1. **CUDA and GPU stacks are large and rarely minimal.** Vendor supplied ML base images bundle substantial toolchains, most of which the inference workload never uses. Every unused component is inventory you must still track and patch.
2. **Models baked into images.** If weights are inside the image, the image is now also a model distribution artifact, and its integrity requirements change accordingly. This connects to the model signing lesson.
3. **Pin base images by digest, not tag.** `python:3.10` moves; `python@sha256:...` does not. A tag reference means your SBOM describes an image you may no longer be running.
4. **Rebuild to patch.** Containers are immutable, so remediation means rebuilding and redeploying, not patching in place. That makes SBOM generation on every build the natural control point.

## 5. Summary

1. A container SBOM inventories the full image: base OS, system libraries, runtime and application dependencies
2. It is broader than an application SBOM, and the base image is usually where the vulnerabilities are
3. Generate with `trivy image` in the build pipeline
4. Containers are how AI models reach production, making this the scope closest to a deployed system
5. Pin base images by digest so the SBOM describes what is actually running
