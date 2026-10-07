# Exercise: Building a Speech to Text System

Course: CAISP (Practical DevSecOps)
Status: Complete

## How to read this document

Two audiences. For the big picture with no code, read **Part 1 (Introduction)** and **Part 5 (Conclusion)**. To replicate the work, read **Part 2 (First Principles)**, **Part 3 (Step by Step)** and **Part 4 (Security Analysis)**, which carry every command and script with explanations.

---

# Part 1: Introduction (for everyone)

## What we are doing

We are building a tool that listens to a recording of someone speaking and writes down what they said. This is called **speech to text**, or Automatic Speech Recognition (ASR). Every voice assistant, automatic subtitle, and call centre transcript relies on it.

## The idea in plain terms

Sound is just air pressure wobbling over time. A microphone turns those wobbles into a long list of numbers. The job of an ASR model is to look at that list of numbers and work out which words the wobbles represent. It has learned to do this by being shown thousands of hours of recordings paired with their correct transcripts, until it could map the shapes of sound to the letters and words that produced them.

The model we use here, Wav2Vec2, was trained on 960 hours of read English audiobooks. Feed it a 16,000 numbers per second recording of clear English speech and it returns a transcript.

## Why this lab is different, and why it matters

Every previous lab in this chapter worked with text or images. This one introduces a new kind of input: **audio**. That matters for security because audio is a new doorway into an AI system. A transcript produced from audio is then often fed onward, into a chatbot, a search box, a command handler, and at that point spoken words have become untrusted text driving a system. Anyone who controls the audio controls that text.

## What to take away

Speech to text converts sound into words that a downstream system then acts on. It is rarely the end of a pipeline; it is the start of one. So the security question is not only "did it transcribe correctly" but "what happens next with whatever it transcribed, and who got to choose the audio". Treat a transcript as untrusted user input, because that is exactly what it is.

---

# Part 2: First Principles

Four principles cover what the code does.

**Principle 1: Audio is numbers sampled over time, and the sampling rate must match the model.**
A WAV file stores sound as a stream of numbers, so many per second (the **sampling rate**). This model expects exactly 16,000 samples per second (16 kHz), because that is what it was trained on. Feed it audio at a different rate and the timing is wrong and the transcription is garbage. This is why the code checks the rate and refuses to proceed if it is wrong.

**Principle 2: An ASR model is not an LLM, and needs its own loading classes.**
Different model families need different transformers classes. A chatbot uses `AutoModelForCausalLM`; this ASR model uses `Wav2Vec2ForCTC` for the model and `Wav2Vec2Processor` for the audio preprocessing. Picking the right pair is the practical skill, and the Hugging Face model page's "Use this model" button tells you which to use.

**Principle 3: The model outputs probabilities per time slice, not words directly.**
For each small slice of audio the model outputs **logits**: a score for every possible character. Taking the highest scoring character in each slice (`argmax`) gives a raw sequence of characters. A decoding step (CTC decoding) then collapses repeats and blanks into readable words. So the pipeline is audio to logits to character IDs to text.

**Principle 4: The transcript is the model's best guess, not ground truth.**
ASR output contains errors, especially on names, technical terms and unusual words. The sample transcripts in this lab prove it: "software" became "soft rare", "pinning" became "penning", "access" became "aciss". Any system consuming a transcript must treat it as approximate.

---

# Part 3: Step by Step Replication

## 3.0 Environment and tools

1. Linux, Python, pip (lab uses plain `pip`, not `uv`, here)
2. `torch==2.6.0`, `transformers==4.30.0`, `soundfile==0.13.1`
3. Model: `facebook/wav2vec2-base-960h` (small, trained/fine tuned on 960 hours of LibriSpeech at 16 kHz)
4. Input: WAV files sampled at 16,000 Hz

## 3.1 Set up dependencies

```bash
apt update && apt install python3-pip -y

mkdir asr-speech-to-text && cd asr-speech-to-text

cat > requirements.txt <<EOF
torch==2.6.0
transformers==4.30.0
soundfile==0.13.1
EOF

pip install -r requirements.txt
```

`torch` runs the model, `transformers` provides the Wav2Vec2 classes, and `soundfile` reads WAV files into numbers.

## 3.2 Load the processor and model

```python
from transformers import Wav2Vec2Processor, Wav2Vec2ForCTC
import soundfile as sf
import torch

revision_id = "22aad52d435eb6dbaf354bdad9b0da84ce7d6156"
processor = Wav2Vec2Processor.from_pretrained("facebook/wav2vec2-base-960h", revision=revision_id)
model = Wav2Vec2ForCTC.from_pretrained("facebook/wav2vec2-base-960h", revision=revision_id)
```

Two objects, each doing a distinct job (Principle 2):

1. **`Wav2Vec2Processor`**: prepares raw audio into the exact numeric format the model expects, and later decodes the model's output back into text. It is the audio equivalent of a tokenizer.
2. **`Wav2Vec2ForCTC`**: the model itself. CTC (Connectionist Temporal Classification) is the technique that lets a model align a long audio stream to a shorter text sequence without being told which slice maps to which letter.

The `revision_id` pins the exact model version, the same supply chain good practice as every other lab.

## 3.3 The transcription function

```python
def speech_to_text(audio_file, sampling_rate=16000):
    audio_file = audio_file.strip().replace("\n", "").replace("\r", "")

    # Load the audio file (Principle 1)
    audio_input, sampling_rate_ = sf.read(audio_file)

    # Refuse audio at the wrong sampling rate (Principle 1)
    if sampling_rate_ != sampling_rate:
        raise ValueError(
            f"Audio file's sampling rate is {sampling_rate_}, but the model expects {sampling_rate} Hz."
        )

    # Prepare the audio for the model
    inputs = processor(audio_input, return_tensors="pt", sampling_rate=sampling_rate, padding=True)

    # Run the model (Principle 3): audio -> logits
    with torch.no_grad():
        logits = model(input_values=inputs.input_values).logits

    # logits -> highest scoring character per slice -> text (Principle 3)
    predicted_ids = torch.argmax(logits, dim=-1)
    transcription = processor.decode(predicted_ids[0])

    return transcription.lower()
```

Walking the pipeline:

1. `sf.read` returns the audio as numbers plus its actual sampling rate. The rate check enforces Principle 1: wrong rate, no transcription.
2. `processor(...)` normalises the audio and packs it into a PyTorch tensor (`return_tensors="pt"`; use `"tf"` for TensorFlow).
3. `model(...).logits` under `torch.no_grad()` produces the per slice character scores. `no_grad` skips gradient bookkeeping, which is only needed for training, making inference faster and lighter.
4. `torch.argmax(logits, dim=-1)` picks the top character in each slice.
5. `processor.decode(...)` collapses that raw sequence into readable text (CTC decoding).
6. `.lower()` returns lowercase, since the model emits uppercase.

## 3.4 The interactive loop

```python
if __name__ == "__main__":
    while True:
        print("+" * 50)
        audio_file = input("\033[92mEnter your WAV file path: Type 'X' or 'x' to exit: \033[0m").strip()
        if audio_file in ['X', 'x']:
            print("Exiting.")
            break
        transcription = speech_to_text(audio_file)
        print("Returned Transcription:", transcription)
```

The same prompt loop pattern as the chatbot and summarizer labs, taking a file path and printing the transcript until the user types x or X.

## 3.5 Get sample audio and run

```bash
git clone https://gitlab.practical-devsecops.training/marudhamaran/caisp-sample-files.git
ls -al caisp-sample-files/audio-samples

python3 simple_speech_to_text.py
# then enter, one at a time:
#   caisp-sample-files/audio-samples/01.wav
#   caisp-sample-files/audio-samples/02.wav   ... up to 09.wav
```

The first transcription takes about a minute (model loading); later ones take seconds.

## 3.6 A note on the startup warning

On load the program prints: "Some weights of Wav2Vec2ForCTC were not initialized ... You should probably TRAIN this model." This is expected and harmless here. The warning fires because the class can host extra layers that this particular checkpoint does not fill, but for straightforward transcription the loaded weights are sufficient. Worth understanding rather than ignoring: the same warning on a model you intend to rely on could mean it genuinely is not ready.

## 3.7 Observed results

The transcripts were accurate in substance but littered with errors on exactly the hard words: "software" to "soft rare", "dependency pinning" to "dependency penning", "access" to "aciss", "network" to "nepwork", "Netflix" to "nafflics". Content was recoverable; precise wording was not. This is Principle 4 in the open.

The lab also hints at an `09.wav` described only as "interesting", suggesting it contains something unusual (accented, noisy, non speech, or an adversarial style sample) worth transcribing to see how the model degrades.

---

# Part 4: Security Analysis

## Vulnerabilities and concerns

1. **The transcript is untrusted input for whatever comes next.** On its own this program only prints text. But ASR almost always feeds a downstream system (a voice assistant executes the words, a support pipeline routes them, an LLM answers them). At that boundary, spoken audio has become text that drives behaviour, and whoever supplied the audio supplied that text. This is the audio on ramp to prompt injection and command injection.
2. **Adversarial audio is a real evasion surface.** Just as TextFooler perturbed text and BadNets perturbed images, audio can be perturbed. Research has shown noise imperceptible or barely perceptible to humans that makes ASR transcribe attacker chosen words, and commands hidden in music or ambient sound. This is the Evade ML Model tactic from the Chapter 2 ATLAS notes, in the audio domain.
3. **Unbounded file input with no validation beyond sampling rate.** `sf.read(audio_file)` opens whatever path the user supplies, with no size limit and no path restriction. A very long recording can exhaust memory (denial of service), and the path is not confined to a working directory. The only check is the sampling rate, which is a correctness gate, not a security one.
4. **The `.strip().replace(...)` on the path is assignment-correct but weak.** Unlike the sentiment lab, here the result IS assigned, so it works, but stripping newlines is not path sanitisation. Traversal and absolute paths pass straight through.
5. **Transcription errors are a safety issue when the output is trusted.** Principle 4's errors matter: if a transcript drives an automated decision (a medical dictation, a spoken command, a compliance record), "access" heard as "aciss" or a negation dropped can change meaning. Confidence is not part of the returned value, so a caller cannot tell a shaky transcript from a solid one.
6. **Model supply chain, again.** The model is downloaded from Hugging Face; the revision is pinned (good), but the same trust considerations as every ASR/LLM lab apply.

## Defences and mitigations

1. **Treat every transcript as untrusted user input.** Whatever consumes the transcript must validate, escape or constrain it exactly as it would text typed by a stranger. Never route a raw transcript straight into a command or a prompt without a boundary.
2. **Validate and bound the audio input.** Restrict file paths to an allowed directory, cap file size and duration before loading, and reject unexpected formats. Enforce the sampling rate (as the lab does) but add the resource controls it lacks.
3. **Defend the ASR stage against adversarial audio** with input preprocessing (denoising, filtering), confidence thresholding, and where stakes are high, human review of low confidence transcripts. Liveness and channel checks help against replayed or synthetic audio.
4. **Surface confidence, do not hide it.** Return a per transcript confidence (derivable from the logits) so downstream logic can refuse or escalate uncertain results instead of trusting them blindly.
5. **Keep model provenance disciplined**: pinned revision, verified source, safe loading.

## Code and strategy improvements

1. **Return confidence alongside text.** Use the softmax of the logits to produce an average confidence, and return it with the transcript so callers can gate on it.
2. **Add resource and path controls.** Cap duration and file size, and restrict the path to a working directory, turning the sampling rate check into a full input validation step.
3. **Handle non 16 kHz audio gracefully.** Rather than only erroring, resample to 16 kHz (the processor and libraries support it), so the tool is usable without failing on the common case of differently sampled files.
4. **Label the downstream boundary.** If this feeds another system, document and enforce that the transcript is untrusted, so the injection risk is handled where the text is consumed.

---

# Part 5: Conclusion (for everyone)

We built a working speech to text tool by loading a pre trained ASR model, feeding it 16 kHz audio, and turning the model's per slice character predictions into a readable transcript. It transcribed clear English well, and stumbled exactly where these systems always do: on names, technical terms and unusual words.

The wider point is about where speech to text sits. It is almost never the finish line; it is the entrance. A transcript becomes the input to something else, and at that moment audio has been converted into text that makes a system act. Two consequences follow. First, the audio itself is an attack surface, susceptible to adversarial perturbation just like text and images earlier in this chapter. Second, and more important, the transcript must be treated as untrusted input by whatever consumes it, because a stranger's voice has just become a stranger's text inside your system.

For a security professional the takeaway is a habit of mind: follow the data past the demo. The lab ends when the transcript prints, but the risk begins with what the transcript does next. Recognising that the interesting boundary is downstream, not on screen, is the skill this exercise really teaches.

---

## Appendix: file created in this exercise

| File | Role |
| ---- | ---- |
| `simple_speech_to_text.py` | Loads Wav2Vec2, reads a 16 kHz WAV, transcribes it to lowercase text in a prompt loop |
