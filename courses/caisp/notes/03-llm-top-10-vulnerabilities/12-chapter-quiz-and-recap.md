# Chapter 3 Quiz and Recap

Answers confirmed, with why each distractor fails. Read alongside the chapter summary as a revision aid.

## Q1. A resume processing LLM manipulated by text embedded in a resume is which vulnerability?

**Indirect prompt injection.**

The attacker never talks to the model. They plant instructions in content the system will later process, and the victim triggers it by running the system normally. Direct injection would mean typing into the prompt. It is not XSS (no browser, no script execution) and not information leakage (that could be a consequence, not the vulnerability class). Indirect is the more dangerous form precisely because it needs no access.

## Q2. (True/False) It is safe to execute LLM generated code without verification because LLMs are trained to produce secure code.

**False.**

Two independent reasons. First, training data includes vulnerable code, so the model reproduces those patterns. Second, and more importantly, generation can be steered by injection, so "the model is well trained" is irrelevant if an attacker controls part of the input. All generated code needs review, testing and validation before execution.

## Q3. Best mitigation against SQL vulnerabilities in an LLM that generates queries from natural language?

**Verify and sanitise the LLM's output before executing the SQL.**

The other three all try to control the *input* side. Rate limiting addresses availability, not query content. Blocklist input filters are defeated by every obfuscation technique in the prompt injection lesson. A guard model reduces injection likelihood but is probabilistic and can be bypassed. Only output verification sits between the generated query and the database, which is the last point where a deterministic check is possible. In practice, pair it with a least privilege database account so that even an approved query cannot drop a table.

## Q4. Rendering LLM output in a browser: which control mitigates XSS?

**Output encode the LLM responses.**

Contextual encoding converts characters such as angle brackets and quotes into HTML entities so injected script cannot execute. Toxicity analysis addresses harmful content, not code execution. Reviewing database actions is the wrong sink. Browser updates are not a control you own. Note that nothing about LLMs changes this remedy: it is the same fix as for any untrusted content rendered in a page.

## Q5. (True/False) RAG documents cannot be a poisoning vector because they are not training data.

**False.**

They are a poisoning vector, but a different one, and the distinction is worth stating precisely. **Training data poisoning alters model weights and is persistent**, removable only by retraining. **Knowledge base or retrieval poisoning injects context at inference time and is reversible** once the malicious document is removed. Same outcome for the user, entirely different detection and remediation. Documents in a RAG store function as the model's memory at query time, so anyone who can write to that store can influence answers.

## Q6. Which is an example of an expensive operation in a Model DoS attack?

**Requesting complex mathematical calculations such as summing all prime numbers up to one billion.**

The defining property is **asymmetry**: a short, cheap request forcing long, expensive computation. Greetings, short translations and small document summaries are all cheap on both sides. Note that expensive operations are particularly dangerous when the model has a calculator or code execution tool, since the cost then lands on that tool rather than on token generation.

## Q7. What is a context window?

**The maximum amount of text, in tokens, that an LLM can process at once.**

It covers **both the prompt and the generated response**. It is not the user interface, not a browser window, and not a memory timeframe, though the third distractor is the interesting one: chat interfaces appear to remember by prepending prior conversation to each new prompt, and all of that consumes context window. So apparent memory is re-sent history, and long conversations grow more expensive every turn even without an attack.

## Q8. A system tricked into processing extremely large files is which vulnerability?

**Model Denial of Service.**

Resource exhaustion through oversized input. Nothing about data integrity (poisoning), dependencies (supply chain) or data exposure (disclosure) is involved. This is the exact failure seen in the summarizer lab, where a large file produced an out of memory crash because no size limit was applied before loading.

## Q9. (True/False) Scanning libraries for CVEs provides complete protection against supply chain risk.

**False.**

CVE scanning covers **known vulnerabilities in dependencies** and nothing else. It does not detect poisoned model weights, backdoored components, compromised training data, malicious code without a published CVE, or a typosquatted package that is malicious by design rather than vulnerable. The torchtriton dependency confusion attack would not have been caught by a CVE scan. Comprehensive defence requires signing, provenance verification, pinning, integrity checks and monitoring.

## Q10. Highest risk for training data poisoning?

**Training on user prompts and inputs.**

User generated content is inherently untrusted and attacker controllable at will. Curated research articles, internal policy documents and Wikipedia all have some curation or access control between the attacker and the corpus. Tay is the standing evidence: a system that learned from Twitter was producing offensive output within about a day.

## Q11. Most immediate sensitive information disclosure risk in an enterprise LLM deployment?

**Allowing employees to use the LLM to optimise internal source code.**

Source code carries trade secrets, business logic, embedded credentials and security implementations. Public marketing materials, public reviews and generic templates contain nothing confidential. This is the Samsung scenario exactly, and the aggravating factor there was the consumer tier trained on submitted conversations. The same activity on an enterprise tier that excludes customer data from training is a materially different risk.

## Q12. Highest risk plugin for sensitive information disclosure?

**A plugin that reads emails, especially password reset emails.**

Email access reaches private correspondence, business secrets and, critically, **password reset messages, which are the key to every other account the user holds**. Public page summarisation, travel suggestions and weather lookups touch no confidential data. This is not hypothetical: email exfiltration through plugin abuse was demonstrated by Rehberger in 2023.

## Q13. Most effective countermeasure against context window exhaustion?

**Setting limits on the tokens a user or endpoint may consume.**

It addresses the mechanism directly and preventively. Backups are recovery, not prevention. A private on premises model still has a finite context window. Monitoring detects the attack after it starts rather than stopping it. In practice, cap input and output separately, since the asymmetry attack is cheap to send and expensive to answer.

## The thread through all thirteen

Reading them together, the quiz tests three things:

1. **Mechanism recognition** (Q1, Q5, Q6, Q8): can you name the vulnerability class from a description of what happened?
2. **Control selection** (Q3, Q4, Q13): can you pick the control that acts on the actual mechanism rather than one that sounds security adjacent?
3. **Realistic risk judgement** (Q10, Q11, Q12): given several plausible options, which actually carries the sensitive data or the untrusted input?

The habit underneath all three: **follow the data and follow the privilege.** Where did this input come from, where is this output going, and what is it permitted to do when it gets there.
