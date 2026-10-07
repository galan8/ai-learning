# Application SBOM

## 1. What it covers

An **application SBOM** is a comprehensive inventory of the software components used in a particular application: proprietary code, open source libraries, frameworks and their transitive dependencies.

This is the narrowest of the three SBOM scopes in this chapter, and the most commonly produced. It answers: **what did we build this application out of?**

## 2. Contents

| Field | Detail |
| ----- | ------ |
| **Name** | The component identifier |
| **Version** | Exact version, which is what CVE matching depends on |
| **Licensing terms** | Obligations attached to use and redistribution |
| **Origin or source** | In house, third party, or pulled from a third party repository |
| **Dependencies** | What this component itself depends on, so transitive risk is visible |
| **Hashes** | Integrity of the specific artifact |

**Transitive dependencies are the point.** Most applications directly declare a few dozen dependencies and actually ship several hundred once transitive ones resolve. The components most likely to hurt you are usually ones no developer chose deliberately.

## 3. Generating one

```bash
# CycloneDX SBOM for a project directory
trivy fs --format cyclonedx --output app-sbom.json /path/to/project

# Scan the produced SBOM for vulnerabilities
trivy sbom app-sbom.json
```

The second command is the part that matters. **An SBOM has no security value until something consumes it.** Generating and filing one is compliance theatre; generating one and continuously matching it against advisories is a control.

## 4. Reading the output

A CycloneDX document has three parts worth knowing:

1. **`metadata`**: what this BOM describes, when it was generated, and by which tool
2. **`components`**: the flat list of everything found, each with name, version, licences, and a `purl` (package URL) used for vulnerability matching
3. **`dependencies`**: the graph showing which component depends on which, which is what lets you distinguish direct from transitive

## 5. In an AI context

An application SBOM for an LLM application captures the framework layer, and that layer is where several documented incidents landed: `transformers`, `torch`, `langchain`, `faiss`, the embedding libraries.

Recall from the LLM Top 10 notes that **the LangChain CVEs (arbitrary code execution via `exec`, SQL injection, SSRF) were vulnerabilities in the glue code, not the model.** Those are exactly the components an application SBOM inventories, which makes this the SBOM scope that would have caught them.

What an application SBOM does **not** capture: the model weights, the training data, the base model lineage. That gap is what the ML-BOM lesson exists to fill.

## 6. Summary

1. Inventory of the components in a single application, including transitive dependencies
2. Name, version, licence, origin, dependencies and hashes
3. Generate with Trivy or equivalent in CI, then scan the SBOM continuously
4. For AI applications it captures the framework layer, where the LangChain class of vulnerability lives
5. It does not capture models or data, which need an ML-BOM
