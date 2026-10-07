# Exercise: Performing Sentiment Analysis Using an LLM

Course: CAISP (Practical DevSecOps)
Status: Complete

## 1. Objective

Build a sentiment analysis program using a three class RoBERTa model, including a preprocessing step that anonymises usernames and URLs before the text reaches the model. This covers a fourth model class (`AutoModelForSequenceClassification`) and shows how raw model outputs (logits) are converted into ranked, human readable scores.

## 2. Environment and tools

1. Lab environment (Linux), Python 3, pip
2. `transformers==4.30.0`, `numpy==1.26.2`, `scipy==1.10.1`
3. Model: `cardiffnlp/twitter-roberta-base-sentiment`, a RoBERTa model trained on tweets, classifying text as negative, neutral or positive
4. Label mapping fetched at runtime from the cardiffnlp TweetEval repository on GitHub

## 3. Model classes so far

The course has now used four `Auto*` classes, and knowing which to pick is the practical skill:

| Class | Purpose | Lab |
| ----- | ------- | --- |
| `AutoModelForCausalLM` | Text generation (decoder only) | Chatbot, RAG |
| `AutoModelForSeq2SeqLM` | Input to output transformation (encoder plus decoder) | Summarizer |
| `AutoModelForSequenceClassification` | Classifying a whole text into categories (encoder only) | This lab, TextAttack |
| `AutoTokenizer` | Text to tokens and back, paired with any of the above | All labs |

The reliable way to find the right one: the "Use this model" button on the Hugging Face model page shows the loading code for that specific model.

## 4. Steps

### 4.1 Imports and model loading

```python
from transformers import AutoModelForSequenceClassification, AutoTokenizer
import numpy as np
from scipy.special import softmax
import csv
import urllib.request

MODEL = "cardiffnlp/twitter-roberta-base-sentiment"
revision_id = "daefdd1f6ae931839bce4d0f3db0a1a4265cd50f"

model = AutoModelForSequenceClassification.from_pretrained(MODEL, revision=revision_id)
tokenizer = AutoTokenizer.from_pretrained(MODEL, revision=revision_id)
```

### 4.2 Loading the label mapping

The model outputs indices (0, 1, 2), not words. The mapping (0 negative, 1 neutral, 2 positive) is downloaded from GitHub at startup:

```python
labels = []
mapping_link = "https://raw.githubusercontent.com/cardiffnlp/tweeteval/main/datasets/sentiment/mapping.txt"
with urllib.request.urlopen(mapping_link) as f:
    html = f.read().decode("utf-8").split("\n")
    csvreader = csv.reader(html, delimiter="\t")
labels = [row[1] for row in csvreader if len(row) > 1]
```

### 4.3 Preprocessing: anonymising usernames and URLs

```python
def preprocess(text):
    new_text = []
    for t in text.split(" "):
        t = "@user" if t.startswith("@") and len(t) > 1 else t
        t = "http" if t.startswith("http") else t
        new_text.append(t)
    return " ".join(new_text)
```

Any word starting with `@` becomes `@user`, and any word starting with `http` becomes `http`. This matches how the model was trained (the TweetEval corpus uses those placeholders), so it improves accuracy as well as stripping identifiers. The lab calls preprocessing optional good practice; in fact matching the training time preprocessing is what keeps predictions reliable.

### 4.4 Sentiment analysis

```python
def analyze_sentiment(text):
    if text is None or text == "":
        return "Exiting: Received empty string or a None object"
    text.strip().replace("\n", "").replace("\r", "")

    text = preprocess(text)
    encoded_input = tokenizer(text, return_tensors="pt")
    output = model(**encoded_input)
    scores = output[0][0].detach().numpy()
    scores = softmax(scores)

    ranking = np.argsort(scores)[::-1]

    results = []
    for i in range(scores.shape[0]):
        results.append((labels[ranking[i]], np.round(float(scores[ranking[i]]), 4)))

    for label, score in results:
        print(f"{label}: {score}")

    positive_score = next((score for label, score in results if label == "positive"), 0)
    if positive_score > 0.5:
        print("The sentiment is positive.")
    return positive_score
```

What each stage does:

1. `tokenizer(text, return_tensors='pt')` converts text to token IDs as a PyTorch tensor
2. `model(**encoded_input)` unpacks the tokenizer output as keyword arguments and runs inference, returning raw logits
3. `.detach().numpy()` removes the tensor from the gradient graph and converts to a NumPy array
4. `softmax` turns logits into probabilities summing to 1
5. `np.argsort(...)[::-1]` sorts ascending then reverses, giving highest score first
6. The results list pairs each label with its rounded score, ranked

### 4.5 Interactive loop

```python
if __name__ == "__main__":
    while True:
        print("+" * 50)
        text = input("\033[92mEnter your text: Type 'X' or 'x' to exit: \033[0m").strip()
        if text in ["X", "x"]:
            print("Exiting.")
            break
        sentiment = analyze_sentiment(text)
        print("Returned Sentiment:", sentiment)
```

## 5. Results

The program prints all three scores ranked, then the returned positive score. A negative example produced roughly 0.83 negative against 0.02 positive, so the ranking and the returned value behaved as expected.

## 6. Bugs and design issues found

1. **The strip and replace on line 3 does nothing.** `text.strip().replace("\n", "").replace("\r", "")` returns a new string that is never assigned. Python strings are immutable, so the original `text` is unchanged and newlines survive into the model. The fix is `text = text.strip().replace(...)`. Since the input loop already calls `.strip()`, the bug is masked in normal use, which is exactly how this kind of defect survives review.
2. **The return value discards most of the information.** The function computes all three scores, prints them, and then returns only the positive score. A caller receiving 0.02 cannot tell whether the text was strongly negative or merely neutral, because negative and neutral collapse into the same low positive value. Returning the full results, or the top label plus its score, would be more useful.
3. **The "greater than 0.5 is positive" rule does not fit a three class model.** With three classes a text can be the clear winner at 0.45 positive while negative and neutral split the rest. The correct approach for multi class output is to take the highest scoring label (`ranking[0]`), which the code already computes and then ignores for this decision.
4. **The username check is naive.** Splitting on spaces alone means `@johndoe,` (with a comma) and any `@` handle followed by punctuation are handled inconsistently, and an email address like `me@example.com` is not caught because it does not start with `@`. Preprocessing intended as anonymisation needs to be more rigorous than this.

## 7. Lessons learned (security focus)

1. **The anonymisation is the security relevant part, and it is incomplete.** Replacing `@handles` and URLs strips two identifier types, but the text still carries names, email addresses, phone numbers, account numbers and any other PII a reviewer wrote. If the motivation is preventing personal data from reaching a model (a real concern when the model is hosted by a third party), this preprocessing gives partial coverage at best. Presenting it as "cleaning" understates what proper de-identification requires.
2. **The label mapping is fetched over the network at every startup.** The model revision is pinned but the mapping is not: a raw GitHub URL is downloaded and its contents are used to label every prediction. If that file changed, or the request were intercepted, the labels would silently reorder and every output would be mislabelled while the model itself worked correctly. It is also a hard runtime dependency on an external repository, so the program breaks offline. Pinning to a commit and caching locally fixes both.
3. **This is the same classifier attack surface as the TextAttack lab, now with three classes.** The model returns full probability distributions, which is what makes adversarial search efficient. Everything learned about evading the DistilBERT classifier applies here, and the extra neutral class gives an attacker a softer target: nudging content into "neutral" may be enough to avoid a moderation trigger without the effort of a full flip.
4. **Twitter trained models carry their training domain with them.** This model learned from tweets, so it handles slang, hashtags and short informal text well and is less reliable on long formal prose. Deploying it outside that domain produces confident but poorly calibrated scores, which matters if downstream decisions are automated.
5. **Sentiment scores driving automated decisions is a risk pattern.** Once a numeric score gates something (flagging reviews, prioritising support tickets, moderating content), anyone who can craft input can influence the outcome. The score looks objective and is not.

## 8. Ideas to take forward

1. Experiment idea: fix the four issues above (assignment bug, return the full ranked results, use `ranking[0]` for the decision, pin and cache the label mapping) as a small hardening exercise
2. Experiment idea: run TextFooler from the previous lab against this three class model and compare how much easier it is to reach "neutral" than to reach a full sentiment flip
3. Experiment idea: feed the same text with and without preprocessing and measure how much the scores move, testing the claim that preprocessing is merely optional
4. Concept file candidate: `concepts/model-classes-in-transformers.md`, a short reference on which `Auto*` class fits which task, drawing on all the labs so far
