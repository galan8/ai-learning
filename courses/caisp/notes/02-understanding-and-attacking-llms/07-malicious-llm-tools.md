# Malicious LLM Tools in the Wild

## XXXGPT

A toolkit advertised for deploying remote access trojans, botnets and other malware. Marketed functions include generating code for various types of malicious software, customising malware to target specific systems, managing and coordinating botnets, and providing tutorials on using the toolkit.

## WormGPT

A generative AI tool sold to cybercriminals for business email compromise, malware and phishing campaigns. It was built on GPT J, an open source model released in 2021, showing that an open weight model plus removal of safety training is enough to produce a criminal service.

## FraudGPT

Another malicious generative AI tool sold on dark web forums and Telegram, used to create content that facilitates cyberattacks. Its main selling point is the absence of any controls preventing it from answering malicious requests.

## The pattern

Across all three, the underlying capability is not exotic: it is a general purpose model with the guardrails stripped away and a criminal user interface bolted on. This is why safety alignment, model release policy and abuse monitoring are security controls, not just ethics topics.
