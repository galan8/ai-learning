# Exercise: Creating a Simple Rule Based Chatbot

Course: CAISP (Practical DevSecOps)
Status: Complete (basic and enhanced versions)

## 1. Objective

Build a rule based chatbot in Python that matches user input against predefined patterns and returns canned responses, then enhance it with regex pattern matching, randomised responses and basic context awareness (remembering the user's name). The point of the exercise is to understand the pre LLM approach to chatbots and its limitations, as a baseline for comparing against the LLM chatbot.

## 2. Environment and tools

1. Lab environment (Linux) with Python 3 and pip preinstalled
2. No third party dependencies: only the Python standard library (`re`, `random`, `time`), so no requirements.txt needed
3. Libraries that a more advanced version might use: `nltk` for NLP, `tensorflow` or `pytorch` for models, `flask` for a web interface

## 3. Steps

### 3.1 Version 1: dictionary matching (`chatbot.py`)

The core is a dictionary mapping expected inputs to fixed responses, and a loop that checks whether any dictionary key appears as a substring of the user's (lowercased) input:

```python
responses = {
    "hello": "Hi there! How can I assist you today?",
    "how are you": "I'm just a program, but I'm functioning as expected!",
    "what is your name": "I'm SimpleBot, a rule-based chatbot.",
    "what can you do": "I can answer simple questions based on predefined patterns.",
    "bye": "Goodbye! Have a great day!",
}

print("=" * 50)
print("Simple Rule-Based Chatbot")
print("Type 'bye' to exit the conversation")
print("=" * 50)

while True:
    user_input = input("\nYou: ").lower()
    if user_input == "bye":
        print("\nChatbot: Goodbye! Have a great day!")
        break
    response = "I'm sorry, I didn't understand that. Can you try asking something else?"
    for pattern in responses:
        if pattern in user_input:
            response = responses[pattern]
            break
    print(f"\nChatbot: {response}")
```

How it works: input is lowercased for case insensitive matching, each dictionary key is tested as a substring, the first match wins, and anything unmatched falls through to a default apology. "How are you doing today?" matches because it contains "how are you"; "Tell me a joke" hits the default.

### 3.2 Version 2: enhanced chatbot (`enhanced_chatbot.py`)

The enhanced version adds four capabilities: regex patterns with word boundaries, multiple randomised responses per pattern, a context dictionary that persists across turns, and special commands.

```python
import re
import random
import time

patterns_responses = [
    (r'\b(hi|hello|hey)\b', ['Hello!', 'Hi there!', 'Hey! How can I help you?']),
    (r'\bhow are you\b', ["I'm doing well, thanks!", "I'm just a program, but I'm functioning perfectly!"]),
    (r'\bwhat is your name\b', ["I'm SimpleBot, a rule-based chatbot.", 'You can call me SimpleBot!']),
    (r'\btime\b', [f"The current time is {time.strftime('%H:%M:%S')}"]),
    (r'\bweather\b', ["I can't check the weather yet, that would require an API integration."]),
    (r'\bjoke\b', ["Why don't scientists trust atoms? Because they make up everything!",
                   'What do you call a fake noodle? An impasta!',
                   'Why did the scarecrow win an award? Because he was outstanding in his field!']),
    (r'\bthank you\b', ["You're welcome!", 'Happy to help!', 'No problem!']),
    (r'\bbye\b', ['Goodbye!', 'See you later!', 'Take care!'])
]

context = {
    'user_name': None,
    'last_topic': None,
    'questions_asked': 0
}

help_message = ("I can respond to greetings, tell jokes, show the time, and answer basic "
                "questions. Try asking me 'what is your name' or 'tell me a joke'!")

print("=" * 50)
print("Enhanced Rule-Based Chatbot")
print("Type 'bye' to exit the conversation")
print("Type 'help' for assistance")
print("=" * 50)

while True:
    user_input = input("\nYou: ")
    context['questions_asked'] += 1
    user_input_lower = user_input.lower()

    if user_input_lower == "bye":
        if context['user_name']:
            print(f"\nChatbot: Goodbye, {context['user_name']}! Have a great day!")
        else:
            print("\nChatbot: Goodbye! Have a great day!")
        break

    if user_input_lower == "help":
        print(f"\nChatbot: {help_message}")
        continue

    if context['user_name'] is None and 'my name is' in user_input_lower:
        name_match = re.search(r'my name is (\w+)', user_input_lower)
        if name_match:
            context['user_name'] = name_match.group(1).capitalize()
            print(f"\nChatbot: Nice to meet you, {context['user_name']}! How can I help you today?")
            continue

    response = "I'm not sure I understand. Type 'help' for assistance."
    for pattern, responses in patterns_responses:
        if re.search(pattern, user_input_lower):
            context['last_topic'] = pattern
            response = random.choice(responses)
            break

    print(f"\nChatbot: {response}")
```

The improvements over version 1:

1. Regex with `\b` word boundaries matches whole words flexibly ("hi", "hello" or "hey" anywhere in the sentence) without false substring hits
2. `random.choice` over multiple responses gives variety instead of identical replies
3. The `context` dictionary is primitive memory: it captures the user's name with `re.search(r'my name is (\w+)')` and personalises the goodbye
4. Special commands (`help`) are checked before pattern matching

## 4. Result

Both versions ran as expected. Version 1 answered its five known phrases and apologised for everything else. Version 2 varied its greetings, told the time, remembered "My name is ..." and said a personalised goodbye. Anything outside the pattern list still fell through to the default response.

## 5. Limitations of rule based chatbots

1. Only recognises exact matches or substrings; any rephrasing outside the patterns fails
2. No understanding of context or meaning behind the words
3. Cannot learn from interactions or improve over time
4. Limited strictly to the responses that were programmed in

Possible further enhancements: intent classification, entity recognition, API integrations (weather, knowledge bases), persistent memory between sessions, and proper NLU.

## 6. Lessons learned (security focus)

1. **Rule based bots are fully deterministic and auditable.** Every possible response exists in the source code, so the security review surface is the code itself. There is no model that can be jailbroken, no hallucination, and no hidden behaviour. This is the trade off the LLM chatbot gives up in exchange for open ended capability.
2. **Even here, user input handling is the sensitive seam.** The name capture regex takes attacker controlled input and stores it in state that is later echoed back into output. Harmless in a terminal, but the same pattern in a web chatbot becomes an injection vector (XSS through the stored name) if output is not escaped.
3. **The time response has a subtle bug worth noticing.** `time.strftime` runs once, when the pattern list is built, not when the user asks. The bot reports the startup time forever. A good reminder that f strings evaluate immediately, and that "works in the demo" is not "works correctly".
4. **The comparison with the LLM chatbot is the real lesson.** Rule based: predictable, safe, cheap, and rigid. LLM based: flexible and fluent, but probabilistic, opaque, and carrying a whole new attack surface (prompt injection, supply chain, hallucination). Most of this course is about the security cost of that jump in capability.

## 7. Ideas to take forward

1. Experiment idea: fix the time bug (make the time response a callable evaluated per request) and add output escaping to the name feature
2. Concept file candidate: `concepts/rule-based-vs-llm-chatbots.md` comparing the two labs side by side
