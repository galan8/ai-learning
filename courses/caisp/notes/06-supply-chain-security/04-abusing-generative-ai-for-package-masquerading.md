# Abusing Generative AI for Package Masquerading

## 1. The attack

A new supply chain attack that did not exist before code generating assistants: **the attacker does not need to trick a developer, because the AI does it for them.**

```
   1. Attacker asks an AI assistant a plausible development question
          "How do I integrate OrientDB with Node.js?"
                          │
                          ▼
   2. Model hallucinates a package name that does not exist
          "npm install orientdb.js"        ← no such package
                          │
                          ▼
   3. Attacker registers that exact name on the public registry
          and publishes a malicious package under it
                          │
                          ▼
   4. A real developer asks the same question, gets the same
      recommendation, and installs the attacker's code
```

Step 2 is the whole vulnerability. The model produced a confident, well formatted answer including install instructions, sample code and a link to the package page. Everything about the response looked correct except that the package did not exist.

**Why the attack works reliably rather than occasionally:** hallucinations are **repeatable**. The USENIX Security 2025 study found that 43% of hallucinated package names recurred across all ten repeated runs of the same prompt. The attacker does not need to guess what a model might invent; they can enumerate what it reliably invents and register those names. This is why it scales.

The term for registering hallucinated names is **slopsquatting**, coined by Seth Larson of the Python Software Foundation in April 2025. It is typosquatting where the LLM makes the typo.

## 2. Related: cache poisoning of the answer

The same mechanism has a second form. If an assistant retrieves from the web or from a cached corpus, poisoning that source poisons the recommendation. The attack surface is not only the model's memory but anything shaping its answer.

## 3. The defence: read the registry metadata

The practical defence, and the one the lesson demonstrates, is that **the legitimate package and the hallucinated one look completely different on the registry page**.

For the OrientDB example, the real driver is `orientjs`, not `orientdb.js`. Its npm page shows exactly the signals a fabricated package cannot fake:

| Signal | Real package (`orientjs`) | What a slopsquatted package looks like |
| ------ | ------------------------- | -------------------------------------- |
| **Version history** | 34 versions | 1, or a handful published at once |
| **Downloads** | ~417 weekly, ~1.4k monthly, with a consistent trend line | Near zero, or a sudden unexplained spike |
| **Dependents** | 53 packages depend on it | None |
| **Publish history** | Last published 8 months ago, long history behind it | Published days or weeks ago |
| **Repository** | Links to a real GitHub org with issues and pull requests | Missing, or a repository with no history |
| **Issues and PRs** | 136 issues, 9 pull requests, evidence of a community | Empty |

**The heuristic:** a legitimate package has a *past*. Version history, dependents, a download trend, an issue tracker with real conversations. A fabricated package has metadata that is either absent or was all created at once.

## 4. Defences beyond eyeballing

Manual inspection does not scale, so:

1. **Verify before installing.** Check that the package exists and that its metadata is plausible, ideally as an automated pre install gate rather than a human habit.
2. **Use an internal proxy or mirror** with an allowlist, so developers cannot pull directly from public registries. This is the same control as for dependency confusion, and it addresses both.
3. **Pin versions and use lockfiles**, so what was reviewed once is what gets installed every time.
4. **Treat AI generated dependency lists as untrusted input**, subject to the same review as any other third party code recommendation.
5. **Scan the dependency tree in CI**, so a package that made it in is caught before it reaches production.
6. **Prefer packages that appear in your existing lockfiles**, since a package you already depend on is a package you have already accepted.

## 5. Why this belongs in the AI supply chain chapter

It is the cleanest example in the whole course of an **AI-specific supply chain attack that requires no access to any AI system.** The attacker does not poison a model, does not need credentials, and does not touch the victim's infrastructure. They exploit a statistical property of a model everyone else is using, and let the victim's own tooling deliver the payload.

## 6. Summary

1. Models invent package names; attackers register them and wait.
2. The attack scales because hallucinations are repeatable: 43% recur across every repeated run.
3. "Slopsquatting" is typosquatting where the model makes the typo.
4. Registry metadata is the tell: version history, dependents, download trend, issue tracker, publish history.
5. A legitimate package has a past; a fabricated one has metadata created all at once.
6. Defence is verification before install, internal mirrors with allowlists, pinning, and treating AI dependency suggestions as untrusted.
