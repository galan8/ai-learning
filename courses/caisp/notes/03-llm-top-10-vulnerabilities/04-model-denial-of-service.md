# Model Denial of Service

## 1. What it is

Model denial of service occurs when an attacker keeps an LLM so busy processing their requests that it becomes unavailable, or prohibitively expensive, for legitimate users.

Traditional DoS remains relevant against LLM infrastructure, but LLMs add their own variants that exploit how the model itself works. In the 2025 OWASP list this sits under **LLM10: Unbounded Consumption** (it was LLM04: Model Denial of Service in the 2023 version). The rename is meaningful: the modern framing covers not just availability but **cost**, since on metered inference an attacker who cannot take the service down can still run up the bill. This is sometimes called denial of wallet.

## 2. Entry points

1. **The inference API**, where prompts arrive directly
2. **The application or mobile front end** that wraps it
3. **External sources the model ingests**, such as large documents submitted for summarisation or long logs passed for analysis

Anything that determines how much work the model does is an entry point.

## 3. Traditional DoS, for context

These predate LLMs and still apply to the servers hosting them.

**Network layer:**

1. **SYN flood**: open many TCP connections with SYN requests and never complete the handshake, exhausting the connection table.
2. **Ping of death**: send malformed or oversized ICMP packets to crash or freeze a target (historically effective against unpatched stacks).
3. **Smurf attack**: send ICMP echo requests to a broadcast address with the victim's spoofed source address, so every host on the network replies to the victim.

**Application layer:**

1. **HTTP flood**: overwhelm the server with a high volume of legitimate looking requests.
2. **Slowloris**: hold many connections open by sending partial HTTP requests very slowly, exhausting worker threads with minimal bandwidth.

**The relevant pattern**: Slowloris is the closest classical analogue to LLM DoS, because it consumes resources through slow, cheap, individually valid requests rather than volume. Most model DoS works the same way: the requests look legitimate, they are just expensive to serve.

## 4. LLM specific DoS methods

### Resource exhaustion through repeated requests

Repeated prompts, each individually reasonable, can consume compute beyond what the deployment can supply. A stream of translation or summarisation requests is a plain example: nothing about any single request is abusive.

### Context window exhaustion

The **context window** is the maximum amount of text, measured in tokens, that a model can process at once. It covers **both the input prompt and the generated response**, and it is a finite resource.

**A necessary detour: does an LLM remember your conversation?**

Not by itself. A direct API call is **stateless**. Sending "My hobby is polo" and then, in a separate call, "What's my hobby?" produces an answer that the model does not know, because nothing carried over between the two requests.

Chat interfaces appear to remember because the application prepends the previous conversation to each new prompt before sending it. The apparent memory is re-sent history, not stored state, and **all of it consumes context window**.

Two consequences follow:

1. A long conversation grows the prompt on every turn, so cost and latency climb even without an attack.
2. An attacker can fill the window deliberately, by submitting a very large prompt or by crafting one that induces an extremely long response. Once full, the model either fails or begins discarding earlier content, which can also drop system instructions from context.

### Expensive operations

Prompts that demand complex reasoning, long chains of explanation, or synthesis across many knowledge domains. For example, a request to explain every stage of manufacturing an autonomous robot from raw material extraction through final delivery, plus the social and economic effects of each stage. Or a request for a complete history of every Olympic event, all sports across summer, winter and Paralympic games, with participant lists per event per year and prize money by country.

The characteristic these share: **a short, cheap prompt that forces a long, expensive response**. The asymmetry is the whole attack. Ten tokens in, thousands of tokens of computation out.

### Complex mathematical questions

Questions requiring heavy computation and many intermediate results, for example asking for the sum of all prime numbers up to one billion. Particularly dangerous where the model has a code execution or calculator tool attached, since the cost then lands on that tool rather than on token generation.

## 5. Why this is harder to filter than classical DoS

Every one of the above is a legitimate looking request. There is no malformed packet, no signature, no forbidden keyword. Distinguishing an abusive prompt from a genuinely demanding one requires reasoning about cost, not about content, which is why the defences below are resource controls rather than content filters.

## 6. Mitigations

**From the lesson:**

1. **Rate limiting** per IP address, per user or per API key, plus network level DoS protection.
2. **Monitor resource utilisation** and respond to abnormal spikes.
3. **Enforce context window limits** to a specified maximum number of tokens.

**Worth adding:**

4. **Cap input and output separately.** Limit prompt size on the way in and `max_tokens` on the way out. Output limits matter more, because the asymmetry attack is cheap to send and expensive to answer.
5. **Set timeouts and queue limits** so a single expensive request cannot occupy a worker indefinitely, and so queues cannot grow without bound.
6. **Validate and bound ingested content.** Cap the size of documents and URLs accepted for summarisation or analysis, which is precisely the gap seen in the summarizer lab, where a large file crashed the process.
7. **Apply quotas and cost alerting** per user and per tenant, since on metered APIs the first symptom of abuse is often a bill rather than an outage.
8. **Restrict expensive tools.** If the model can call code execution or external APIs, rate limit and sandbox those paths separately.

## 7. Summary

1. Model DoS makes an LLM unavailable or uneconomic by keeping it busy with attacker work.
2. Now framed by OWASP as unbounded consumption, covering cost as well as availability.
3. Classical network and application DoS still apply to the hosting infrastructure; Slowloris is the closest analogue in spirit.
4. LLM specific methods: repeated requests, context window exhaustion, expensive reasoning prompts, and heavy computation.
5. The context window covers prompt plus response, and chat memory is re-sent history, so conversations consume it continuously.
6. The core asymmetry is a cheap prompt forcing an expensive response.
7. Defence is resource control: rate limits, token caps in both directions, timeouts, input size limits, quotas, monitoring and cost alerting.
