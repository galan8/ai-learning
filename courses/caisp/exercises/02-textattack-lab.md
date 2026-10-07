# Exercise: Attacking an LLM Model using TextAttack

Course: CAISP (Practical DevSecOps)
Status: Complete (classifier built, TextFooler attack run over 20 examples, results analysed)

## 1. Objective

Build a sentiment analysis classifier, then attack it with TextAttack to generate adversarial examples: inputs modified just enough to flip the model's prediction while (ideally) preserving the original meaning to a human reader. This is the first offensive lab of the course, and it demonstrates the Defense Evasion tactic from MITRE ATLAS in practice.

## 2. Environment and tools

1. Lab environment (Linux), Python 3.10 virtual environment, dependencies installed with `uv`
2. `torch==2.6.0`, `transformers==4.49.0`, `datasets`, `nltk`, `scikit-learn`, TensorFlow stack, `textattack==0.3.10`
3. Target model: `distilbert-base-uncased-finetuned-sst-2-english` (DistilBERT fine tuned on the Stanford Sentiment Treebank, binary positive/negative)
4. Attack recipe: TextFooler (Jin et al., 2019)

## 3. Background

**Sentiment analysis** determines the emotional tone of text. Real uses include customer feedback analysis, market research, social media monitoring and brand monitoring. Note the model class: this is an encoder model doing classification (BERT family), not a generative model, which fits the BERT versus GPT distinction from the Chapter 2 notes.

**TextAttack** is a Python framework for adversarial attacks, data augmentation and model training in NLP. An attack succeeds when a small change to the input flips the model's prediction while keeping the text semantically similar. TextFooler works by identifying the words that most influence the prediction and swapping them for candidate synonyms until the label changes.

## 4. Steps

### 4.1 Build the classifier

```python
import torch
from transformers import DistilBertTokenizer, DistilBertForSequenceClassification
from torch.nn.functional import softmax

model_name = "distilbert-base-uncased-finetuned-sst-2-english"
revision = "714eb0fa89d2f80546fda750413ed43d93601a13"

device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
tokenizer = DistilBertTokenizer.from_pretrained(model_name, revision=revision)
model = DistilBertForSequenceClassification.from_pretrained(model_name, revision=revision)
model = model.to(device)
model.eval()

def predict_sentiment(text):
    inputs = tokenizer(text, return_tensors="pt", truncation=True, padding=True)
    inputs = {k: v.to(device) for k, v in inputs.items()}
    with torch.no_grad():
        outputs = model(**inputs)
        predictions = softmax(outputs.logits, dim=1)
    negative_prob = predictions[0][0].item()
    positive_prob = predictions[0][1].item()
    label = "POSITIVE" if positive_prob > negative_prob else "NEGATIVE"
    confidence = positive_prob if label == "POSITIVE" else negative_prob
    return {"label": label, "confidence": confidence,
            "probabilities": {"negative": negative_prob, "positive": positive_prob}}
```

Unlike the earlier image classifier, this one exposes probabilities via `softmax`, which is what makes the attack possible: the attacker can see how close the model is to flipping.

Baseline testing on ten clean examples gave correct labels with confidence around 0.99 throughout.

### 4.2 Wrap the model for TextAttack

```python
from textattack.models.wrappers import ModelWrapper
from sentiment_classifier import predict_sentiment, model

class SentimentWrapper(ModelWrapper):
    def __init__(self):
        self.model = model

    def __call__(self, text_inputs):
        outputs = []
        for text in text_inputs:
            result = predict_sentiment(text)
            outputs.append([result["probabilities"]["negative"],
                            result["probabilities"]["positive"]])
        return torch.tensor(outputs)
```

TextAttack needs a uniform interface: give it a list of strings, get back a tensor of `[negative_prob, positive_prob]` per string. Any model behind any API can be attacked once wrapped this way.

### 4.3 Run the attack

```python
from textattack.attack_recipes import TextFoolerJin2019
from textattack import Attacker, AttackArgs
from textattack.datasets import Dataset

model_wrapper = SentimentWrapper()
dataset = Dataset(examples)        # 20 (text, label) pairs, label 0 negative / 1 positive

attack = TextFoolerJin2019.build(model_wrapper)
attack_args = AttackArgs(num_examples=20, log_to_csv="attack_results.csv",
                         checkpoint_interval=5, disable_stdout=False)
attacker = Attacker(attack, dataset, attack_args)
results = attacker.attack_dataset()
```

Setup issue encountered: the first run failed with an NLTK `LookupError` for `averaged_perceptron_tagger_eng`. Fixed by running `nltk.download('averaged_perceptron_tagger_eng')` in a Python shell. Not a code error; TextFooler needs a part of speech tagger to pick grammatically valid substitutions.

## 5. Results

TextAttack's own summary over 20 examples:

| Metric | Value |
| ------ | ----- |
| Successful attacks | 19 |
| Failed attacks | 1 |
| Original accuracy | 100.0% |
| Accuracy under attack | 5.0% |
| Attack success rate | 95.0% |
| Average perturbed word % | 20.94% |
| Average words per input | 9.8 |
| Average queries per example | 78.05 |

A model with perfect accuracy on clean input dropped to 5% accuracy under attack, at the cost of changing roughly one word in five and about 78 queries per example.

Representative flips:

1. "The worst movie I've ever seen." to "The finest movie I've ever seen." Negative to positive with one word changed
2. "I really enjoyed watching this film." to "I really rained watching this film." Positive to negative, one word
3. "I think this product is nice and works well for my needs." to "...works better for my needs." Positive to negative, one word

## 6. Critical reading of the results

The lab highlights example 12 ("works well" to "works better") as its favourite, calling it a semantic preserving adversarial attack. That framing is worth interrogating, and so is the headline success rate.

1. **Most of these substitutions do not preserve meaning.** "worst" to "finest" reverses the meaning entirely, so the model's new label is arguably correct. "enjoyed" to "rained", "superb" to "unpaid", "struggled" to "triumphed", "waste" to "jingles" all change or destroy meaning. A genuine adversarial example fools the model while a human would still read the original sentiment. By that standard, only a handful here qualify, with "works well" to "works better" being the strongest, since a human reads no sentiment change at all.
2. **TextFooler optimises against a word embedding similarity metric, not human judgement.** Words that sit close in embedding space are treated as synonyms even when they are not. This is why "rained" can be selected as a substitute for "enjoyed". The 95% figure measures success against that metric, so it overstates real world attack quality.
3. **The custom analysis in the lab's own code contradicts the summary table.** Re-running `predict_sentiment` on the perturbed text showed several attacks flagged successful where the sentiment did not actually flip (examples 6, 7, 8, 16, 17, 18, 19, 20 printed the "attack marked as successful but sentiment did not flip" warning). Some flipped only marginally (example 7 dropped positive confidence from 0.9998 to 0.5805 without crossing the boundary). Taking the re-check at face value, the true flip rate is closer to half than to 95%, so the reported figure should be read as TextAttack's internal accounting, not as verified outcomes.
4. **Non determinism does not explain the discrepancy.** The lab attributes output variation to the LLM being non deterministic, but this classifier runs `model.eval()` under `torch.no_grad()` with a deterministic softmax, so the same text yields the same probabilities. The variation comes from the attack search, which explores different candidate substitutions, not from the model.

The right takeaway is not "95% of attacks succeed" but "an automated tool with only query access reduced a confident classifier to near zero accuracy on its own terms, and a meaningful minority of those attacks are genuinely meaning preserving". That is still a serious result.

## 7. Lessons learned (security focus)

1. **This is the Evade ML Model technique from MITRE ATLAS, executed end to end.** The Chapter 2 notes described crafting input to deceive a model while preserving harmful functionality. Substitute a spam filter or toxicity classifier for the sentiment model and the same technique moves abusive content past moderation intact.
2. **Confidence scores are what make the attack efficient.** The attacker used about 78 queries per example, guided by probability changes after each substitution. Returning only the label, or rounding and rate limiting confidence outputs, makes this search far more expensive. Exposing raw probabilities through an API is a real design decision with security consequences.
3. **High clean accuracy says nothing about robustness.** The model was 100% accurate on the test set and 5% accurate under attack. Accuracy and robustness are separate properties, and only the first is usually measured. This is the same lesson as the fine tuning lab's perfect validation score, in a sharper form.
4. **Query patterns are detectable.** Dozens of near identical inputs with single word variations arriving from one source is a recognisable signature. Rate limiting, per client anomaly detection and logging of near duplicate queries are practical defences that do not require changing the model.
5. **Defences exist and belong in the notes.** Adversarial training (including adversarial examples in the training data), input preprocessing such as spell and grammar checking to catch nonsense substitutions, ensembling multiple models, and confidence thresholding with human review for low confidence cases. None makes a model immune; each raises attacker cost.
6. **Verify attack results independently.** The lab's own re-check disagreed with the framework's success counter. Any security tool that reports its own success rate should be validated against ground truth before those numbers go into a report.
7. **The same tooling is dual use.** TextAttack is a legitimate robustness testing framework: the ability to attack your own classifier before an adversary does is exactly what makes it valuable defensively.

## 8. Ideas to take forward

1. Experiment idea: re-run the attack with a wrapper that returns only the label, or heavily rounded probabilities, and compare query counts and success rate to quantify how much confidence exposure helps the attacker
2. Experiment idea: filter the successful attacks by whether a human would agree the meaning is preserved, and report that as the honest success rate
3. Experiment idea: try a different recipe (for example a character level attack such as DeepWordBug) and compare how the perturbations look and how they might be defended against
4. Concept file candidate: `concepts/adversarial-examples.md` covering evasion attacks, the meaning preservation criterion, and why robustness differs from accuracy
5. Concept file candidate: `concepts/model-api-exposure.md` on what information an inference API leaks (confidence scores, logits, timing) and how that enables evasion, extraction and membership inference
