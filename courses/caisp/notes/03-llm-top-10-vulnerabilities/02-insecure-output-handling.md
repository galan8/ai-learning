# Insecure Output Handling

## 1. What it is

Insecure (or improper) output handling is the failure to validate, sanitise and constrain what an LLM produces **before passing it to another component**. It stems from a single mistaken assumption: that the model's response can be trusted.

The core principle: **an LLM's output is untrusted input to whatever consumes it.** Because the output is influenced by the prompt, and the prompt can be influenced by an attacker, letting model output flow unchecked into a browser, shell, database or API effectively grants users indirect access to those systems.

**A note on the OWASP numbering.** The course calls this the second highest risk, which reflects the original 2023 list where it was **LLM02: Insecure Output Handling**. In the current 2025 revision it is **LLM05: Improper Output Handling**, and LLM02 is now Sensitive Information Disclosure. Same vulnerability, renumbered and renamed. Quote whichever version you are working against, but know both.

## 2. Why it multiplies other risks

Insecure output handling rarely causes harm on its own; it is the mechanism by which other Top 10 risks become real incidents. If you trust everything the model sends your way:

1. **Excessive agency**: model output triggers actions beyond what the user should be able to perform.
2. **Sensitive information disclosure**: private, confidential or personal data flows out through an unchecked response.
3. **Remote code execution** on systems that consume LLM output.
4. **Misinformation and toxic content** reaching users unfiltered, undermining trust in the system itself.

The relationship worth holding: **prompt injection is the way in, improper output handling is the way out.** An injection that reaches nothing does nothing. It becomes an exploit at the point where output crosses into a system that acts on it.

**Distinguish it from overreliance.** Improper output handling concerns what happens to output *before* it reaches downstream components. Overreliance concerns humans depending too heavily on the accuracy of output they have already received. Different problem, different fix.

## 3. What can go wrong, by destination

The vulnerability class depends entirely on where the output lands:

| Output flows into | Resulting vulnerability |
| ----------------- | ----------------------- |
| Browser (HTML, JavaScript, Markdown) | Cross site scripting (XSS), CSRF |
| System shell, `exec` or `eval` | Remote code execution |
| SQL query without parameterisation | SQL injection, destructive queries |
| File path construction | Path traversal |
| Outbound HTTP request | Server side request forgery (SSRF), data exfiltration |
| Email or document template | Phishing, formula injection |

## 4. Worked example: the XSS loop

A development team builds an LLM tool that explains shell and code commands. A user submits a command containing a script tag, for example a snippet that raises a JavaScript alert. The model does its job perfectly and returns an explanation that includes the original snippet. The application renders that explanation into the page without encoding it. The browser executes the script.

Three things make this instructive:

1. **The model was not compromised.** It answered exactly as designed. The failure is entirely in how the answer was consumed.
2. **The payload round trips.** Attacker input goes into the model, comes back inside a legitimate looking explanation, and is rendered as trusted content. Explanation and documentation tools are especially exposed because reproducing user supplied text is their whole purpose.
3. **The fix is old and well understood.** Contextual output encoding before rendering. Nothing about LLMs changes the remedy.

## 5. Documented incidents: LangChain

**CVE-2023-29374 (arbitrary code execution).** In LangChain up to version 0.0.131, the `LLMMathChain` component passed model generated Python to `exec()`. A researcher showed that asking the "calculator" to evaluate an expression that imported the `os` library could read environment variables, exposing the OpenAI API key. Rated CVSS 9.8 (critical), classified under CWE-74, improper neutralisation of special elements in output used by a downstream component. Fixed in 0.0.142.

That CWE is worth reading twice: the weakness is defined in terms of *output used by a downstream component*, which is exactly this chapter's topic, and it is the same CWE family as SQL and command injection.

**Destructive SQL through chained database access.** LangChain components that let a model query a database have repeatedly been shown to generate destructive statements, including dropping tables, when the database credentials permit it. Related LangChain CVEs from the same period include SQL injection issues and an SSRF in a URL loader. The pattern is consistent: the model produces a plausible query, the framework executes it, and nothing in between asks whether that query should be allowed.

**The general lesson from LangChain's CVE history**: the vulnerabilities were not in the models. They were in the glue code that took model output and handed it to `exec`, to a database, or to a URL fetcher without a checkpoint.

## 6. Scope discipline when testing

When testing an LLM application, establish its intended purpose and hold it to that purpose, especially where output is consumed by another system. A model built to explain commands should not be able to produce output that deletes databases, attacks browsers, or reads sensitive operating system files. The question is not only "did it answer correctly" but "what is the worst thing this answer could do to the system that receives it".

## 7. Mitigations

Ask, before consuming any output: **where is this going, and what could it do there?**

1. **Encode output for its destination context.** HTML encode before rendering in a browser; escape appropriately for shell, SQL, or template contexts. Match the encoding to the sink.
2. **Constrain database access.** Review which actions are permitted before executing model generated SQL. Use a read only account, parameterised queries, and an allowlist of permitted operations. Never grant DDL rights to a path a model can influence.
3. **Treat outbound URLs with caution.** Any URL the model produces or fetches is a potential exfiltration channel for chat history and personal or confidential data, and a potential SSRF. Use allowlists and block internal address ranges.
4. **Screen responses for toxicity and sensitive content** before presenting them to a user.
5. **Never pass output to `exec` or `eval`.** If code execution is genuinely required, sandbox it with no credentials, no network and no filesystem access.
6. **Apply zero trust to the model.** OWASP's guidance is to treat the model as you would any other user: validate its output, and enforce least privilege on everything it can reach.
7. **Log and monitor outputs**, with rate limiting and anomaly detection, so abuse patterns are visible.

## 8. Summary

1. Model output is untrusted input to whatever consumes it.
2. Originally LLM02 (2023), now LLM05 in the 2025 OWASP list; the course's "second highest" reflects the older numbering.
3. Injection is the entry, improper output handling is what turns it into an incident.
4. The vulnerability that results is determined by the destination: browser gives XSS, shell gives RCE, database gives SQL injection, outbound request gives SSRF.
5. The model behaving correctly is not a defence, as the XSS explanation loop shows.
6. LangChain's CVE history shows the flaw lives in the glue code, not the model.
7. Fixes are conventional application security: encode for context, parameterise, least privilege, sandbox, allowlist, monitor.
