# Exercise: Using Ollama for Running AI Models Locally

Course: CAISP (Practical DevSecOps)
Status: Complete (install, pull, CLI, API, Flask app, Modelfiles), with worked solutions for every practice exercise

## How to read this document

For the concept with no code, read **Part 1** and **Part 5**. To reproduce the lab, work through **Part 3**. **Part 3.8** has worked solutions to all the practice exercises. **Part 4** is the security analysis, and it is the part that matters most: this lab teaches you to run an unauthenticated inference server and, in its last step, to expose it to the network, so understanding that exposure is the real lesson.

This is a Chapter 2 foundational lab (running models), but it reaches forward: the Modelfile `SYSTEM` line is the same steering, not enforcement, idea as the system prompts lab, and `ollama pull` is a supply chain action like the ones the trojan and signing labs are about.

---

# Part 1: Introduction (for everyone)

## What we are doing

We install Ollama, a tool that runs large language models on your own machine instead of calling a cloud service, then pull a few small models, talk to them from the command line and over a local API, wrap one in a small web app, and finally bake custom personas into models using configuration files. By the end you can run private AI locally and build applications on top of it.

## The idea in plain terms

Normally, using an AI model means sending your text to a company's servers. Ollama flips that: it downloads the model's weights to your computer and runs the model there, so your data never leaves the machine. It starts a small local web server (on port 11434) that your programs talk to, the same way they would talk to a cloud API, except it is on your own box. You can also write a short configuration file (a "Modelfile", modelled on Docker's Dockerfile) to give a model a fixed persona and settings.

## Why it matters

1. **Privacy and control.** Sensitive data can be analysed without handing it to a third party, which is genuinely useful in security work.
2. **It is an inference server, and servers have attack surface.** The moment you run Ollama you have a network service with no password on it. That is fine locked to your own machine and dangerous the moment it is opened to a network, which the last step of this lab does.
3. **Local does not mean safe.** Running a model yourself moves the responsibility for securing it, patching it, and judging its output onto you.

## What to take away

Ollama makes local AI genuinely easy, and that ease hides two sharp edges: the API has no authentication, and open models have weaker safety limits than hosted ones. Run it bound to your own machine, put real authentication in front of anything that must be reachable over a network, keep the runtime patched, and check what the model tells you before you act on it.

---

# Part 2: First Principles

**Principle 1: The model runs locally, but it runs as a server.** Ollama starts a background service exposing a REST API at `http://localhost:11434`. Every CLI command and every app talks to that server. By default it listens only on `localhost` (127.0.0.1), which is what keeps it private.

**Principle 2: The API has no authentication.** There is no built in login, token, or access control on that server. Whoever can reach the port can list, run, pull, create, and delete models and consume all the machine's compute. Security comes entirely from who can reach the port, which is why the localhost default matters so much.

**Principle 3: A model is pulled from a registry, so it is a supply chain artifact.** `ollama pull phi` downloads weights from the Ollama registry, the same trust decision as `pip install` or a Hugging Face download. You are trusting the publisher and the registry.

**Principle 4: A Modelfile is configuration as code, and its `SYSTEM` line is a baked in system prompt.** It sets a persistent persona and parameters. As the system prompts lab showed, a system prompt steers behaviour; it does not enforce a boundary and is not a safety control, even when it says "emphasise responsible disclosure".

**Principle 5: Model parameters trade determinism for creativity.** `temperature`, `top_p`, and `top_k` control how random the output is. Low temperature gives focused, repeatable answers; high temperature gives varied, creative, less reliable ones. This is the same sampling lever from the Chapter 2 chatbot labs, now exposed as tunable settings.

---

# Part 3: Step by Step Replication

## 3.1 Install Ollama

```bash
curl -fsSL https://ollama.com/install.sh | OLLAMA_VERSION=0.21.0 sh

apt update
apt install -y ca-certificates
```

Verify it installed and the server is up:

```bash
ollama --version                       # ollama version is 0.21.0
curl http://localhost:11434/api/tags   # {"models":[]}  (empty until you pull one)
```

The empty `{"models":[]}` confirms the server is running and answering on port 11434.

Ollama stores models and config under `~/.ollama` (Linux and macOS). A couple of useful environment variables:

```bash
export OLLAMA_MODELS=/usr/share/ollama/.ollama/models   # where model weights are stored
```

**Flag on the lab's env var list.** The lab also shows `OLLAMA_GPU_MEMORY` and `OLLAMA_CPU_ONLY`. These are not official Ollama environment variables and will have no effect. The reliable ones are `OLLAMA_HOST` (what address and port to bind, which matters a lot for security, see Part 4), `OLLAMA_MODELS` (storage path), and `OLLAMA_KEEP_ALIVE` (how long a model stays loaded). To force CPU only, set `CUDA_VISIBLE_DEVICES=""` before starting the server, not `OLLAMA_CPU_ONLY`. Check the official docs for the current list rather than trusting the lab here.

## 3.2 Pull and run your first model

```bash
ollama pull phi            # downloads the Phi model weights (about 1.6 GB)
ollama list                # phi:latest   e2fd6321a5fe   1.6 GB
ollama run phi "Explain quantum computing in simple terms"
```

The model answers in a few seconds (after a one time load into memory). A second example:

```bash
ollama run phi "Write a short poem about artificial intelligence"
```

**Flag on the content.** Phi's quantum computing answer contains a confident factual error: it claims superposition allows "the creation of parallel universes of computation". That is a pop science misconception, not how quantum computing works. It is a clean, unprompted example of a small model hallucinating authoritatively, which is exactly why the lab's own intro warns you to validate model outputs before relying on them.

Multi turn interactive chat (type `/bye` or Ctrl+D to exit):

```bash
ollama run phi
>>> Hello, who are you?
>>> What can you do to help with cybersecurity?
```

See which models are loaded, and stop one:

```bash
ollama ps                  # shows loaded models and memory use
ollama stop phi
```

**Flag on the CLI table.** The lab's command table lists `ollama kill` to stop a model, but that is not a real command; the correct one is `ollama stop` (which the lab itself uses here). Also, `ollama ps` may report a larger size (for example 5.4 GB) than `ollama list` (1.6 GB), because `list` shows the weights on disk while `ps` shows the memory footprint once the model and its context are loaded.

Compare a second model on the same prompt:

```bash
ollama pull gemma:2b
ollama run gemma:2b "Explain quantum computing in simple terms"
```

Gemma's answer is better structured (the dimmer switch analogy) than Phi's. Comparing small models on the same prompt is the point here.

## 3.3 Interact using the CLI (advanced)

Interactive multi turn sessions keep context within the session:

```bash
ollama run mistral
```

Pipe a prompt in from another command:

```bash
echo "Generate a list of 5 common passwords" | ollama run phi
```

(The model returns the usual worst passwords, `123456`, `password123`, and so on. In a security context this is the list of passwords to forbid, not to use.)

Batch process a file of prompts:

```bash
cat > prompts.txt << EOF
Explain XSS attacks
Explain CSRF attacks
Explain SQL injection
EOF

cat prompts.txt | while read prompt; do
  echo "PROMPT: $prompt"
  echo "RESPONSE: $(ollama run phi "$prompt" 2>/dev/null)"
  echo "-------------------"
done
```

Note a reliability quirk visible in the lab's output: asked about XSS first, the model answered about CSRF, then drifted across all three topics out of order. Small models do not reliably stay on the exact question, which matters if you script them.

## 3.4 Bake a persona with a Modelfile

A Modelfile sets a base model plus a system prompt:

```bash
cat > modelfile-pentest << 'EOF'
FROM mistral
SYSTEM You are a cybersecurity expert specializing in penetration testing. Always format your responses with markdown and include practical examples.
EOF

ollama create pentest-expert -f modelfile-pentest
ollama run pentest-expert
```

The three building blocks: `FROM` picks the base model, `PARAMETER` sets options, `SYSTEM` defines the persistent system prompt. This is the same system prompt idea from the dedicated system prompts lab, now stored in the model itself rather than passed each call.

## 3.5 Use the REST API

The API lives at `http://localhost:11434/api`. Main endpoints: `/api/generate` (one shot text), `/api/chat` (multi turn), `/api/embeddings` (vectors), `/api/tags` (list), `/api/pull` (download).

One shot generation:

```bash
curl -X POST http://localhost:11434/api/generate -d '{
  "model": "phi",
  "prompt": "Explain how a firewall works in simple terms",
  "stream": false
}'
```

Set `"stream": true` to receive the answer in chunks as it is generated.

Multi turn chat, where you pass the whole message history each time (the server is stateless, so the history is how context is carried):

```bash
curl -X POST http://localhost:11434/api/chat -d '{
  "model": "mistral",
  "messages": [
    { "role": "system", "content": "You are a helpful cybersecurity assistant." },
    { "role": "user", "content": "What are the OWASP Top 10?" }
  ]
}'
```

Embeddings (a vector of numbers representing the meaning of the text, useful for search and similarity):

```bash
curl -X POST http://localhost:11434/api/embeddings -d '{
  "model": "phi",
  "prompt": "How to implement strong encryption"
}'
```

## 3.6 A Python API client

```python
cat > ollama_client.py << 'EOF'
import requests
import json
import sys

def generate_text(model, prompt, temperature=0.7):
    """Generate text using the Ollama API."""
    url = "http://localhost:11434/api/generate"
    data = {
        "model": model,
        "prompt": prompt,
        "temperature": temperature,
        "stream": False
    }
    response = requests.post(url, json=data)
    return response.json()["response"]

def chat(model, messages):
    """Have a conversation using the Ollama API."""
    url = "http://localhost:11434/api/chat"
    data = {
        "model": model,
        "messages": messages,
        "stream": False
    }
    response = requests.post(url, json=data)
    return response.json()["message"]["content"]

def list_models():
    """List available models."""
    url = "http://localhost:11434/api/tags"
    response = requests.get(url)
    return [model["name"] for model in response.json()["models"]]

if __name__ == "__main__":
    print("Available models:")
    models = list_models()
    for i, model in enumerate(models):
        print(f"{i+1}. {model}")

    if models:
        model_idx = int(input("\nSelect a model (enter number): ")) - 1
        model = models[model_idx]

        print("\nChoose mode:")
        print("1. Generate text")
        print("2. Chat")
        mode = int(input("Enter choice: "))

        if mode == 1:
            prompt = input("\nEnter prompt: ")
            print("\nGenerating response...\n")
            response = generate_text(model, prompt)
            print(response)
        elif mode == 2:
            messages = [{"role": "system", "content": "You are a helpful assistant."}]
            print("\nChat mode (type 'exit' to quit)")
            while True:
                user_input = input("\nYou: ")
                if user_input.lower() == 'exit':
                    break
                messages.append({"role": "user", "content": user_input})
                print("\nThinking...\n")
                response = chat(model, messages)
                print(f"Assistant: {response}")
                messages.append({"role": "assistant", "content": response})
    else:
        print("No models available. Pull a model first using 'ollama pull <model>'")
EOF

python3 ollama_client.py
```

The chat mode accumulates `messages` and resends the whole list each turn, which is how it keeps context against a stateless server.

## 3.7 A Flask web app (read Part 4 before running it on any network)

```bash
pip install flask --ignore-installed blinker
```

(The `--ignore-installed blinker` works around environments that ship an old `blinker` installed by distutils, which plain pip cannot cleanly replace.)

```python
cat > flask_ollama_app.py << 'EOF'
from flask import Flask, request, jsonify, render_template_string
import requests

app = Flask(__name__)

HTML_TEMPLATE = """
<!DOCTYPE html>
<html>
<head>
    <title>Ollama Chat</title>
    <style>
        body { font-family: Arial, sans-serif; max-width: 800px; margin: 0 auto; padding: 20px; }
        #chat-container { height: 400px; overflow-y: auto; border: 1px solid #ccc; padding: 10px; margin-bottom: 10px; }
        #user-input { width: 80%; padding: 8px; }
        #send-button { padding: 8px 15px; }
        .user-message { background-color: #e6f7ff; padding: 8px; border-radius: 5px; margin: 5px 0; }
        .assistant-message { background-color: #f0f0f0; padding: 8px; border-radius: 5px; margin: 5px 0; }
    </style>
</head>
<body>
    <h1>Ollama Chat</h1>
    <div id="chat-container"></div>
    <div>
        <input type="text" id="user-input" placeholder="Type your message...">
        <button id="send-button">Send</button>
    </div>
    <script>
        const chatContainer = document.getElementById('chat-container');
        const userInput = document.getElementById('user-input');
        const sendButton = document.getElementById('send-button');

        let messages = [
            {"role": "system", "content": "You are a helpful cybersecurity assistant."}
        ];

        function addMessage(role, content) {
            const messageDiv = document.createElement('div');
            messageDiv.className = role + '-message';
            messageDiv.textContent = content;
            chatContainer.appendChild(messageDiv);
            chatContainer.scrollTop = chatContainer.scrollHeight;
            messages.push({"role": role, "content": content});
        }

        function sendMessage() {
            const content = userInput.value.trim();
            if (content) {
                addMessage('user', content);
                userInput.value = '';
                fetch('/chat', {
                    method: 'POST',
                    headers: {'Content-Type': 'application/json'},
                    body: JSON.stringify({messages: messages})
                })
                .then(response => response.json())
                .then(data => { addMessage('assistant', data.response); })
                .catch(error => {
                    console.error('Error:', error);
                    addMessage('assistant', 'Sorry, there was an error processing your request.');
                });
            }
        }

        sendButton.addEventListener('click', sendMessage);
        userInput.addEventListener('keyup', function(event) {
            if (event.key === 'Enter') { sendMessage(); }
        });
    </script>
</body>
</html>
"""

@app.route('/')
def index():
    return render_template_string(HTML_TEMPLATE)

@app.route('/chat', methods=['POST'])
def chat():
    data = request.json
    response = requests.post(
        'http://localhost:11434/api/chat',
        json={
            "model": "mistral",
            "messages": data['messages'],
            "stream": False
        }
    )
    return jsonify({"response": response.json()["message"]["content"]})

if __name__ == '__main__':
    app.run(host='0.0.0.0', debug=False, port=80)
EOF

python3 flask_ollama_app.py
```

It serves a chat page and relays messages to the local Ollama server. Keeping `debug=False` is correct (Flask's debugger is a remote code execution risk if left on). But `host='0.0.0.0'` binds it to every network interface, which is the exposure problem Part 4 is about. On the isolated lab box this is contained; do not copy this binding to anything reachable by others.

---

## 3.8 Worked solutions to the practice exercises

### Step 3: compare phi and mistral on "buffer overflow"

```bash
ollama pull mistral
ollama run phi     "Explain how a buffer overflow works in simple terms"
ollama run mistral "Explain how a buffer overflow works in simple terms"
```

What to compare: mistral (7B) generally gives a longer, more accurate, better structured explanation; phi (2.7B) is faster and shorter but more likely to oversimplify or slip in an error. The exercise is really about noticing that model size and training shape quality, so you choose the right size for the job.

### Step 4, exercise 1: ask one question to several models, save to separate files

```bash
cat > ask-all-models.sh << 'EOF'
#!/bin/bash
# Usage: ./ask-all-models.sh "your question"
QUESTION="$1"
if [ -z "$QUESTION" ]; then
  echo "Usage: $0 \"question\""
  exit 1
fi
mkdir -p responses
for model in phi mistral gemma:2b; do
  safe_name="${model//:/_}"          # gemma:2b becomes gemma_2b for the filename
  echo "Querying $model ..."
  ollama run "$model" "$QUESTION" > "responses/${safe_name}.txt" 2>/dev/null
done
echo "Saved responses in the responses/ directory."
EOF
chmod +x ask-all-models.sh
./ask-all-models.sh "Explain the principle of least privilege"
```

### Step 4, exercise 2: generate JSON and format it with jq

Plain prompts rarely yield clean JSON, so use the API's `"format": "json"`, which constrains output to valid JSON:

```bash
curl -s http://localhost:11434/api/generate -d '{
  "model": "phi",
  "prompt": "List three OWASP Top 10 risks. Reply ONLY with a JSON array of objects, each with name and description fields.",
  "format": "json",
  "stream": false
}' | jq -r '.response' | jq .
```

The first `jq -r '.response'` pulls the model's answer (itself a JSON string) out of the API envelope, and the second `jq .` pretty prints it.

### Step 4, exercise 3: a simple Bash chat interface

```bash
cat > chat.sh << 'EOF'
#!/bin/bash
MODEL="${1:-phi}"
echo "Chatting with $MODEL (type 'exit' to quit)"
while true; do
  read -rp "You: " message
  [ "$message" = "exit" ] && break
  [ -z "$message" ] && continue
  echo -n "Assistant: "
  ollama run "$MODEL" "$message"
  echo
done
EOF
chmod +x chat.sh
./chat.sh mistral
```

Limitation worth knowing: each `ollama run` is independent, so this version has no memory between turns. To keep context, accumulate the history and use the `/api/chat` endpoint (as the Python client in 3.6 does). This is a good illustration of why real chat apps resend the whole conversation.

### Step 5, exercise 1: save conversations to a file (modify the Python client)

Add near the top of `ollama_client.py`:

```python
from datetime import datetime
TRANSCRIPT = "conversation_log.txt"

def log_turn(role, content):
    with open(TRANSCRIPT, "a") as f:
        f.write(f"[{datetime.now().isoformat()}] {role.upper()}: {content}\n")
```

Then, inside the chat loop, log each turn:

```python
            messages.append({"role": "user", "content": user_input})
            log_turn("user", user_input)
            response = chat(model, messages)
            print(f"Assistant: {response}")
            messages.append({"role": "assistant", "content": response})
            log_turn("assistant", response)
```

Every exchange is appended with a timestamp, giving you an audit trail of the session.

### Step 5, exercise 2: let the Flask app choose the model

Three changes to `flask_ollama_app.py`:

1. Add a route that lists models, so the page can populate a dropdown:

```python
@app.route('/models')
def models():
    r = requests.get('http://localhost:11434/api/tags')
    return jsonify([m["name"] for m in r.json()["models"]])
```

2. Add a `<select id="model-select"></select>` to the HTML and fill it on load:

```javascript
fetch('/models').then(r => r.json()).then(list => {
    const sel = document.getElementById('model-select');
    list.forEach(name => {
        const o = document.createElement('option');
        o.value = name; o.textContent = name; sel.appendChild(o);
    });
});
```

3. Send the chosen model with each message and use it server side:

```javascript
body: JSON.stringify({messages: messages, model: document.getElementById('model-select').value})
```

```python
@app.route('/chat', methods=['POST'])
def chat():
    data = request.json
    model = data.get('model', 'mistral')          # fall back to mistral
    response = requests.post(
        'http://localhost:11434/api/chat',
        json={"model": model, "messages": data['messages'], "stream": False}
    )
    return jsonify({"response": response.json()["message"]["content"]})
```

### Step 5, exercise 3: embeddings and similarity

A dedicated embedding model is the right tool, so pull one first:

```bash
ollama pull nomic-embed-text
pip install requests
```

```python
cat > embed_similarity.py << 'EOF'
import requests, itertools, math

OLLAMA = "http://localhost:11434/api/embeddings"
MODEL = "nomic-embed-text"

texts = [
    "How to prevent SQL injection",
    "Protecting databases from malicious queries",
    "Baking a chocolate cake",
]

def embed(text):
    r = requests.post(OLLAMA, json={"model": MODEL, "prompt": text})
    return r.json()["embedding"]

def cosine(a, b):
    dot = sum(x * y for x, y in zip(a, b))
    na = math.sqrt(sum(x * x for x in a))
    nb = math.sqrt(sum(y * y for y in b))
    return dot / (na * nb)

vectors = {t: embed(t) for t in texts}
for a, b in itertools.combinations(texts, 2):
    print(f"{cosine(vectors[a], vectors[b]):.3f}  |  {a}  vs  {b}")
EOF
python3 embed_similarity.py
```

The two security sentences score high similarity to each other and low similarity to the cake sentence, which is the whole idea of embeddings: meaning becomes geometry, so "close in meaning" becomes "close in vector space". (The lab used `phi` for embeddings, which works, but a model trained for embeddings like `nomic-embed-text` gives better quality.)

### Step 6, exercise 1: a PCI DSS compliance auditor

```bash
cat > modelfile-pci-auditor << 'EOF'
FROM mistral
PARAMETER temperature 0.2
PARAMETER top_p 0.9
SYSTEM You are a PCI DSS compliance auditor. For any system or practice described, map it to the relevant PCI DSS v4.0 requirements, state whether it is compliant, non compliant, or needs more information, cite the requirement number, and give one concrete remediation step. Be precise and conservative, and when unsure say so rather than guessing.
EOF
ollama create pci-auditor -f modelfile-pci-auditor
ollama run pci-auditor "We store full card numbers in plaintext in a MySQL database to make refunds easier."
```

Low temperature keeps compliance answers consistent. (Treat its citations as a draft to verify, not as authoritative; see Part 4 on validating output.)

### Step 6, exercise 2: a threat modeling assistant (creative parameters)

```bash
cat > modelfile-threat-modeler << 'EOF'
FROM mistral
PARAMETER temperature 0.7
PARAMETER top_p 0.95
PARAMETER top_k 50
SYSTEM You are a threat modeling facilitator. Given a system description, brainstorm diverse attack scenarios using STRIDE as the lens (Spoofing, Tampering, Repudiation, Information disclosure, Denial of service, Elevation of privilege). Favour breadth and novel angles, and for each threat suggest one mitigation.
EOF
ollama create threat-modeler -f modelfile-threat-modeler
ollama run threat-modeler "A web app with a login page, a file upload feature, and an admin dashboard."
```

Higher temperature widens the range of scenarios, which suits brainstorming. This connects directly to the Chapter 5 threat modeling lab and STRIDE.

### Step 6, exercise 3: a security awareness trainer

```bash
cat > modelfile-sec-trainer << 'EOF'
FROM mistral
PARAMETER temperature 0.5
SYSTEM You are a security awareness trainer for non technical staff. Explain each concept in plain language with a relatable analogy, one real world example, and three concrete do's and don'ts. Avoid jargon, and if you must use a technical term, define it in one sentence.
EOF
ollama create sec-trainer -f modelfile-sec-trainer
ollama run sec-trainer "Explain phishing to new employees."
```

### Step 6, exercise 4: compare parameter configurations

```bash
cat > compare-params.sh << 'EOF'
#!/bin/bash
PROMPT="Describe three risks of weak passwords."
for temp in 0.1 0.7 1.2; do
  echo "=== temperature $temp ==="
  curl -s http://localhost:11434/api/generate -d "{
    \"model\": \"mistral\",
    \"prompt\": \"$PROMPT\",
    \"options\": {\"temperature\": $temp},
    \"stream\": false
  }" | jq -r '.response'
  echo
done
EOF
chmod +x compare-params.sh
./compare-params.sh
```

Run it a few times. At `0.1` the answers are nearly identical run to run (focused and repeatable); at `1.2` they vary and wander (creative but less reliable). That is the determinism versus creativity tradeoff made visible, the same lever as `do_sample` in the Chapter 2 transformers labs.

---

# Part 4: Security Analysis

## The headline: an unauthenticated API, and a step that exposes it

Ollama's REST API on port 11434 has no authentication and no authorization. Anyone who can reach that port can run inference, pull new models, create models, and delete models, and consume all of the machine's CPU, GPU, and memory. The only thing protecting it by default is that Ollama binds to `localhost` (127.0.0.1), so only the local machine can reach it.

Two common actions throw that protection away:

1. **Setting `OLLAMA_HOST=0.0.0.0`** to use Ollama "over the network" binds the unauthenticated API to every interface, open to anyone who can route to the host.
2. **The Flask app in this lab**, `flask_ollama_app.py`, binds `host='0.0.0.0'` on port 80 with a comment that says "so it's accessible externally". That publishes an open, unauthenticated, unrate limited LLM chat endpoint to the network. There is no login, no input validation, no output filtering, and the model is hardcoded. On the isolated lab box this is contained; shipped anywhere real it is an open door.

This is not theoretical. Security researchers have repeatedly found thousands of Ollama servers exposed directly to the internet, free for anyone to use or abuse.

## Real vulnerabilities in the runtime

Running Ollama means running a network service that has had serious bugs:

1. **CVE-2024-37032, "Probllama"** (Wiz Research, June 2024, CVSS 8.8): a path traversal in the model pull endpoint allowed arbitrary file write and, from there, remote code execution, triggerable against an exposed server with a single crafted API request. Patched in Ollama 0.1.34.
2. **"More Models, More ProbLLMs"** (Oligo Security, 2024): six further vulnerabilities in Ollama, several reachable through the same unauthenticated API.

The lab's version, 0.21.0, is patched against those specific issues. The durable lesson is not "upgrade past 0.1.34"; it is that an inference server is an attack surface, new bugs in it keep appearing, and an exposed, unauthenticated instance turns each one into a remote compromise. Patch it, and do not expose it.

## "Local" is a privacy property, not a security property

The lab rightly sells local models on privacy, control, and cost. The tradeoffs it understates:

1. **Weaker safety limits.** Open models like these have lighter safety training than hosted frontier models, and there is no provider side abuse monitoring. The pentest and vuln-expert personas produce exploitation guidance readily. That is useful for legitimate security work and also means the guardrails are yours to add.
2. **You own the whole stack.** Patching the runtime, securing the endpoint, and judging output quality all move to you. Keeping data off a third party's servers does not make the box it runs on secure.
3. **Model provenance still matters.** `ollama pull` fetches from a registry, so you are trusting a publisher exactly as in the trojan and signing labs. Ollama uses the GGUF format, which (unlike pickle) does not execute arbitrary code on load, so this is safer than the pickle supply chain, but "pull only models you trust, from sources you trust" still holds.

## The Modelfile SYSTEM is steering, not a control

The `SYSTEM` line in every Modelfile here is a baked in system prompt. As the system prompts lab showed, that shapes behaviour but does not enforce anything. "Emphasise responsible disclosure" in the vuln-expert persona is a suggestion the model usually follows, not a safety boundary, and not something to rely on if the model is exposed to untrusted users.

## Validate the output

Phi confidently explained quantum computing with a flat falsehood ("parallel universes of computation"). The lab's own intro warns to validate outputs before using them in security critical contexts, and this is why: a small local model will state wrong things fluently, and a compliance citation or an incident response step from one of these personas is a draft to check, never an authority to act on blindly.

## Mitigations, in priority order

1. **Keep Ollama bound to localhost.** Do not set `OLLAMA_HOST=0.0.0.0` on any machine reachable by others. If remote access is genuinely needed, put it behind an authenticating, TLS terminating reverse proxy, or reach it over an SSH tunnel; never expose 11434 or a wrapper directly.
2. **Add the controls the Flask app lacks.** Authentication, rate limiting, input validation, and output filtering belong in front of any app wrapping the model, both to stop abuse and to cap resource consumption (the Model DoS and Unbounded Consumption concern from the OWASP labs).
3. **Patch the runtime** and watch Ollama security advisories; an inference server needs patching like any other service.
4. **Verify model provenance.** Pull only trusted models from trusted sources.
5. **Treat Modelfile SYSTEM prompts as steering, and enforce real limits in code.**
6. **Validate model output** before acting on it, especially for anything security critical.

## Where this sits in the course

This foundational lab quietly touches half the chapters ahead: the Modelfile `SYSTEM` prompt (system prompts lab), model provenance via `ollama pull` (trojan and signing labs), the unauthenticated exposed endpoint (Model DoS and Unbounded Consumption), and hallucinated output (the reliability thread running throughout). Running models locally is easy; running them securely is the actual skill.

---

# Part 5: Conclusion (for everyone)

We installed a tool that runs AI models on our own machine, talked to them from the command line and a small program, wrapped one in a web page, and gave models fixed personalities with short configuration files. It is genuinely easy and genuinely useful: your data stays on your machine, there are no API bills, and you can tailor a model to a task in a few lines.

That ease hides two sharp edges. The first is that the thing you just started is a web server with no password on it. Left on your own machine it is private and fine. The moment anyone opens it to a network, and the lab's final step does exactly that, it becomes an open AI service anyone can drive, on a piece of software that has had serious remote takeover bugs. Researchers keep finding thousands of these sitting open on the internet. The second edge is that running a model yourself does not make it trustworthy: these smaller open models will tell you wrong things with total confidence, and the persona you give them is a costume, not a safety lock.

So enjoy local AI, and run it like the network service it is. Keep it on your own machine, put a real lock in front of anything that has to be reachable, keep it patched, pull only models you trust, and check what it tells you before you believe it. Private is not the same as secure, and this lab is where you learn the difference.

---

## Ideas to take forward

1. Experiment: from a second machine on the same network, try `curl http://<box-ip>:11434/api/tags` first with Ollama on the default localhost binding (refused) and then with it on `0.0.0.0` (answered). Feel the exposure directly, safely, in the lab.
2. Experiment: put a tiny authenticating reverse proxy (nginx with basic auth, or an SSH tunnel) in front of Ollama and confirm the direct port is closed while the proxied path works. The right way to do remote access.
3. Experiment: add rate limiting and a maximum prompt length to the Flask app and show that a flood of requests no longer exhausts the box. Concrete Unbounded Consumption mitigation.
4. Concept file: `concepts/running-models-locally.md` on the privacy versus security distinction, the unauthenticated API, and safe remote access patterns.
5. Concept file: extend `concepts/model-supply-chain.md` with `ollama pull` and GGUF as another model intake path, noting GGUF does not execute code on load unlike pickle.

## Sources

1. Ollama CVE-2024-37032 "Probllama" (Wiz Research, remote code execution via path traversal, patched in 0.1.34): https://www.wiz.io/blog/probllama-ollama-vulnerability-cve-2024-37032
2. Oligo Security, "More Models, More ProbLLMs" (six further Ollama vulnerabilities): https://www.oligo.security/blog/more-models-more-probllms
3. Ollama documentation (API, environment variables, Modelfile): https://docs.ollama.com/
4. OWASP Top 10 for LLM Applications (Unbounded Consumption, Supply Chain): https://genai.owasp.org/llm-top-10/
