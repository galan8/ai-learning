# Natural Language Processing (NLP)

NLP is a branch of computer science and AI that uses machine learning to allow computers to understand, interpret, generate and interact with human language (text and speech).

## The four key steps

1. **Lexical analysis**: breaking the text into tokens (words, numbers, punctuation) and identifying the type of each token, such as article, adjective, noun, verb, adverb or number
2. **Syntactic analysis (parsing)**: checking grammar and building a sentence structure (parse tree) showing how the words relate to each other
3. **Semantic analysis**: extracting the meaning of the sentence, including who did what to whom and how, and relating it to real world understanding
4. **Output formation**: producing the final result (an answer, translation, classification, summary or generated text) based on all the previous steps

## Worked example: "The black cat quickly caught two mice."

1. Lexical analysis: each word is tokenised and tagged. "The" (article), "black" (adjective), "cat" (noun), "quickly" (adverb), "caught" (verb), "two" (number), "mice" (noun), "." (punctuation).
2. Syntactic analysis: a grammar tree is built. Subject: "the black cat" (article, adjective, noun). Predicate: "quickly caught two mice" (adverb, verb, number, noun).
3. Semantic analysis: the action is "caught", performed by the cat on the object "two mice"; "black" modifies the cat and "quickly" describes how the action happened. The system now understands the real world event, not just the words.
4. Output formation: the understanding is used for the task at hand, for example translating the sentence, answering "who caught the mice?" or classifying its sentiment.

## Additional pipeline steps

Pre processing (lowercasing, removing stop words, stemming or lemmatisation), named entity recognition (people, places, dates) and pragmatic analysis (interpreting meaning from context, tone or intent).

## Real world use cases

Grammar and spell checking, machine translation, sentence and text completion (autocomplete), text analytics and sentiment analysis, chatbots and virtual assistants, speech recognition, document summarisation and search engines. Large Language Models (LLMs) such as GPT are the current state of the art in NLP.

**Security relevance:** because NLP systems act on untrusted human input, they are exposed to attacks such as prompt injection, adversarial text that flips a classification (for example bypassing spam or toxicity filters), and data leakage through model outputs.
