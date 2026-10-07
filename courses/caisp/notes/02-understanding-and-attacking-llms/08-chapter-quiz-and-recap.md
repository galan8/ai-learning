# Chapter 2 Quiz and Recap: Understanding and Attacking LLMs

This closes out Chapter 2. Each question is confirmed with the correct answer, a short note on why the other options fail, and just enough context to make it a revision aid as well as an answer key. Read top to bottom and it doubles as a chapter summary.

## Before the questions: the one idea holding the chapter together

An LLM does not understand text; it predicts the next token from statistical patterns learned in training. Everything in this chapter follows from that. Because there is no built in notion of truth, models hallucinate. Because there is no built in separation between instructions and data, they can be injected. Because they are shaped entirely by training data, they can be poisoned. The questions below trace that thread from how models work to how they are attacked.

---

## Part A: How LLMs work

**Q1. Which is NOT a step in how LLMs work? Tokenizing / Embedding / Transforming / Compressing**

Correct: **Compressing**.

The pipeline is tokenizing (text to tokens), embedding (tokens to meaning carrying vectors), transforming (attention layers predict the next token). Compressing is not a stage. Tokenizing, embedding and transforming are all genuine steps, which is exactly why they are the wrong answers to a "NOT" question: the odd one out is the one that never appears in the pipeline.

**Q6. (True/False) During fine tuning, the pre trained weights remain fixed to preserve general understanding.**

Correct: **False**.

Fine tuning updates the pre trained weights, using a low learning rate so the general knowledge is nudged rather than overwritten. The statement describes a different technique (freezing weights, as in feature extraction), so it is false as written. Key distinction to hold: fine tuning changes the model's weights; RAG changes what the model sees at inference time without touching weights.

---

## Part B: Choosing the right approach

**Q3. Best architecture to help a search engine understand queries? GPT / BERT / RL models / Unsupervised models**

Correct: **BERT**.

BERT is bidirectional (encoder only), reading a word's full left and right context, which is what query understanding needs. GPT is decoder only and built to generate text, not to understand it, so it is the plausible trap here. RL models learn from reward signals (games, robotics) and unsupervised models find structure in unlabelled data; neither targets query comprehension. The encoder (understand) versus decoder (generate) split is the thing being tested.

**Q4. Healthcare org with a pre trained model needs accurate medical info. Best approach? New foundational model / Fine tune on medical data / RAG only / More parameters**

Correct: **Fine tune with domain specific medical data**.

Fine tuning adapts existing general knowledge to the medical domain cheaply. Replacing the model throws away what already works; adding parameters does not add medical knowledge and needs full retraining; "RAG only" is the strongest distractor, since in practice RAG plus fine tuning is often best, but on its own RAG supplies documents without teaching the model medical language and reasoning. Given the question's framing (adapt the existing model), fine tuning is the intended answer. Worth noting for the real world: many production medical systems combine both.

**Q5. What does RAG primarily solve? Lower training compute / Better sentence structure / Keeping models current without retraining / Privacy**

Correct: **Keeping models up to date with real world knowledge without retraining**.

A base model has a fixed knowledge cutoff; RAG retrieves current external information at query time and grounds the answer in it. It does not reduce training cost, does not improve grammar (the base model already handles fluency), and does not inherently protect privacy (if anything, connecting a model to private data sources adds privacy exposure, per the chapter's risk notes).

**Q8. Analysing customer feedback across platforms. Best capability? Text Generation / Text Understanding with Sentiment Analysis / Conversational / Information Extraction only**

Correct: **Text Understanding with Sentiment Analysis**.

The task is comprehending and categorising existing feedback, which is understanding, not generation. Text generation creates new content; conversational is for dialogue; "Information Extraction only" is too narrow, capturing facts but missing the emotional tone that feedback analysis is really about. Sentiment analysis is the capability that matches the job.

---

## Part C: Attacking AI systems

**Q2. Designing an LLM code generation system prioritising security and accuracy. Most significant risk? Privacy / Bias / Compute cost / Insecure or vulnerable code**

Correct: **Generation of insecure or vulnerable code**.

For a code generation tool, the defining risk is code that looks functional but carries flaws (injection, poor input validation, weak crypto) that ship into production. Privacy, bias and compute are real concerns but general to any LLM use; only insecure code is specific to and most consequential for this task. The lesson: AI generated code needs the same review and scanning as any untrusted code.

**Q9. Which is NOT one of MITRE ATLAS's 14 tactics? ML Model Access / ML Attack Staging / ML Model Optimization / Exfiltration**

Correct: **ML Model Optimization**.

Model optimisation is a legitimate engineering practice, not an attack tactic. The other three are real ATLAS tactics. The distractor works by pairing plausible ML sounding words with the ATLAS naming style; the tell is that optimisation improves a model rather than compromising one.

**Q10. Why store ML models in protected internal or cloud environments rather than on end user devices? Inference speed / Reduce full model exposure / Bandwidth / Easier updates**

Correct: **To reduce the risk of full model exposure**.

A model on a user's device can be extracted, reverse engineered or copied (the Full ML Model Access technique). Central storage keeps architecture and weights behind access controls. Speed, bandwidth and update convenience may be side benefits but are not the security rationale, which is what the question asks for.

**Q7. (True/False) With internet connected RAG, sources are controlled by a central authority that maintains data quality.**

Correct: **False**.

Nobody vets the open web. RAG over internet sources can pull in unverified or attacker controlled content, which is precisely the indirect prompt injection and data quality problem from the chapter. The consumer must add source verification, filtering and validation. This decentralisation is both RAG's strength (diverse, current information) and its danger (poisoned or low quality content).

**Q11. (True/False) A backdoored model can pass validation because it only misbehaves on specific trigger inputs.**

Correct: **True**.

This is the defining property of a backdoor and the reason clean accuracy is useless for detecting one: the model behaves normally on everything except the attacker's trigger, so standard testing sees a healthy model. Detecting it needs specialised techniques (trigger reconstruction, activation analysis, targeted repair), not more accuracy testing. This ties directly to the BackdoorBox lab.

---

## After the questions: how the eleven fit together

Read in order, the quiz is a map of the chapter:

1. **Q1, Q6** are the mechanism: how a model processes text, and how fine tuning adjusts it.
2. **Q3, Q4, Q5, Q8** are design judgement: pick the right architecture (BERT vs GPT), the right adaptation method (fine tuning vs RAG), and the right capability (understanding vs generation) for a given goal.
3. **Q2, Q7, Q9, Q10, Q11** are the security core: insecure generated code, unvetted RAG sources, the ATLAS tactic taxonomy, protecting model files from extraction, and the stealth of backdoors.

The single sentence that carries all of it: because an LLM predicts plausible tokens rather than knowing truth, every design choice about it is also a security choice, and every capability it has is also an attack surface. That is the chapter.

## One question the quiz does not ask, worth answering yourself

If a model only predicts likely next tokens, why does telling a RAG system "answer only from the context" fail to guarantee it will? Answering that in your own words (the instruction is a soft statistical nudge, not an enforced control, so the model can and does override it) is the best single check that Chapter 2 has landed. It is a strong candidate for a `concepts/` note.
