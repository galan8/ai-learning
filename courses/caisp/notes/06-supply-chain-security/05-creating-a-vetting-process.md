# Creating a Vetting Process for Software Components

## 1. What vetting is

A vetting process ensures **rigorous evaluation and risk management of components from procurement through deployment**. It is the organisational answer to the question the previous lessons raise: given that you inherit everything upstream, what do you actually do before accepting a component?

Vetting is a **framework**, not a tool. Tools implement steps within it.

## 2. The process

```
   1. Define security requirements
              │
              ▼
   2. Initial assessment ──────► licences, integrity checksums,
              │                   maintainer reputation, provenance
              ▼
   3. Code review ─────────────► manual or peer review, plus SAST
              │
              ▼
   4. Functional testing ──────► expected behaviour
              │
              ▼
   5. Runtime testing (DAST) ──► vulnerabilities in execution
              │
              ▼
   6. Software Composition ────► map ALL third party dependencies,
      Analysis (SCA)             including transitive
              │
              ▼
   7. Threat modeling ─────────► entry points, attack surface,
              │                   realistic threat scenarios
              ▼
   8. Ongoing monitoring ──────► vulnerability management, threat
              │                   intelligence, behavioural baselines
              ▼                   and anomaly detection
   9. Compliance, supplier ────► certifications, vendor agreements,
      and supply assessment      service level terms after disclosure
              │
              ▼
  10. Documentation ───────────► results, mitigation actions,
                                  audit trail
```

**Steps 1 and 2 are where most of the value is**, and where most organisations start at step 3. Defining what you require before evaluating, and checking integrity and licences at intake, prevents more than any amount of later scanning.

**Step 6, SCA, deserves emphasis** because it is the step that addresses the statistic from lesson 1: six of seven vulnerabilities come from transitive dependencies. A review that examines only direct dependencies misses most of the exposure.

**Step 8 is what makes it a process rather than an event.** A component vetted in January is not vetted in June. Continuous monitoring, behavioural baselines and anomaly detection are what maintain the assurance.

## 3. Training and awareness

The lesson closes on the point that is easiest to skip and hardest to substitute: **people need to know why this exists and how to respond.** Secure coding practices, vulnerability management, the purpose of vetting, and specifically awareness of dependency confusion and AI hallucinated packages, because those two are defeated by developer habit more than by tooling.

## 4. Tools

Examples in this space include WhiteSource (now Mend), Black Duck, Snyk, Veracode, Xanitizer and Fossa, plus code insight platforms. Tool names change constantly through acquisition; **the process outlives the tooling**, which is why the framework matters more than the vendor list.

## 5. Applying it to AI components

The same ten steps, with different content:

| Step | AI equivalent |
| ---- | ------------- |
| Define requirements | Acceptable licence terms, provenance requirements, permitted model sources |
| Initial assessment | Model card review, licence (open weight vs open source), publisher verification, revision pinning |
| Code review | Review any custom code the model ships, since `trust_remote_code` executes it |
| Functional testing | Behavioural evaluation against your actual use case, not just published benchmarks |
| Runtime testing | Adversarial testing, prompt injection resistance, model scanning |
| SCA | ML-BOM covering base models, datasets and adapters, plus the framework dependency tree |
| Threat modeling | The Chapter 5 approach applied to the AI architecture |
| Monitoring | Watch for model updates, revoked models, new CVEs in the serving stack |
| Compliance | Data provenance and licensing, which carry legal exposure |
| Documentation | ML-BOM, attestations, and a record of what was verified |

**The gap in most organisations:** models are pulled by name from a public hub with none of these steps applied, while a JavaScript library going into the same product goes through all ten.

## 6. Summary

1. Vetting is a framework spanning procurement to deployment, not a scanning tool.
2. Ten steps: requirements, assessment, code review, functional testing, runtime testing, SCA, threat modeling, monitoring, compliance, documentation.
3. Most value sits in defining requirements and assessing at intake, which most teams skip.
4. SCA matters because transitive dependencies carry most of the exposure.
5. Continuous monitoring is what makes it a process rather than a one time event.
6. Training closes the loop, particularly for dependency confusion and hallucinated packages.
7. The same framework applies to models, and is almost never applied to them.
