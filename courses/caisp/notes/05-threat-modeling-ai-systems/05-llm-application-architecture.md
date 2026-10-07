# An LLM Application Architecture

The reference architecture used for the STRIDE exercise. Two views of the same system: a simple component view, and the same system redrawn as a proper DFD with trust boundaries.

## View 1: simple component architecture

```
        ┌──────────────────────────────┐
        │ User / Attacker              │   external entity
        │ (External Entity)            │
        └──────────────┬───────────────┘
                       │
                       ▼
        ┌──────────────────────────────┐
        │ API Gateway / Proxy          │
        │ (Request Routing)            │
        └──────────────┬───────────────┘
                       │
                       ▼
        ┌──────────────────────────────┐        ┌──────────────────┐
        │ Security Authentication      │        │ Logs             │
        │ System (AuthN and Access     │        │ (Data Store)     │
        │ Control)                     │        └────────▲─────────┘
        └──────────────┬───────────────┘                 │
                       │                                 │
                       ▼                                 │
        ┌──────────────────────────────┐                 │
        │ AI Model (Inference /        │─────────────────┘
        │ Decision)                    │
        └──────────────┬───────────────┘
                       │
                       ▼
        ┌──────────────────────────────┐
        │ Data Layer (User Data)       │
        └──────────────┬───────────────┘
                       │
                       ▼
        ┌──────────────────────────────┐        ┌──────────────────┐
        │ Model Training Pipeline      │───────►│ Training Data    │
        │ (Training Process)           │        │ (Data Store)     │
        └──────────────────────────────┘        └──────────────────┘
```

The two components highlighted as the crown jewels in the course slides are the **AI Model (Inference)** and the **Data Layer (User Data)**.

## View 2: the same system as a DFD with trust boundaries

This is the version to threat model against, because it shows where trust changes.

```
                     ┌──────── DMZ ─────────┐   ┌─ Internal Network: ─┐
                     │                      │   │  Interface to DMZ   │
 ┌──────────┐        │   ╭──────────────╮   │   │   ╭─────────────╮   │
 │  User    │        │   │  Security    │◄──┼───┼───┤  AI Model   │◄──┼──┐
 │ (entity) │        │   │  AuthN and   │   │   │   │ (Inference) │   │  │
 └────▲──┬──┘        │   │  Access Ctrl │───┼───┼──►╰──┬───▲──┬───╯   │  │
      │  │           │   ╰───▲──────┬───╯   │   └──────┼───┼──┼───────┘  │
      │  ▼           │       │      │       │          │   │  │          │
 ╭──────────╮        │       │      ▼       │   ┌─ Internal Network: ─┐  │
 │ Browser  │◄───────┼───────┴──╮ ╭─────────┼─┐ │      Protected      │  │
 │  / App   │───────►│  ╭───────┴─┴──────╮  │ │ │   ═══════════════   │  │
 ╰──────────╯        │  │ API Gateway /  │  │ └─┼──►│ Data Layer  │   │  │
                     │  │ Proxy (Routing)│  │   │   │ (User Data) │   │  │
                     │  ╰────────────────╯  │   │   ═══════════════   │  │
                     └──────────────────────┘   │   ═══════════════   │  │
                                                │   │    Logs     │◄──┼──┘
                                                │   ═══════════════   │
                                                └─────────────────────┘

                     ┌──── Internal Network: Development ────┐
                     │  ═══════════════      ╭─────────────╮ │
                     │  │ Training    │─────►│ Model       │ │
                     │  │ Data        │      │ Training    │─┼──► AI Model
                     │  ═══════════════      │ Pipeline    │ │    (Inference)
                     │                       ╰─────────────╯ │
                     └───────────────────────────────────────┘

  Legend:  ╭──╮ process    │ │ external entity    ═══ data store
```

## Reading the boundaries

Four trust zones, and the crossings between them are where the threats are:

1. **Untrusted (the user and their browser).** Everything here is attacker controllable.
2. **DMZ.** API gateway and the authentication and access control system. First point of contact, so spoofing and denial of service concentrate here.
3. **Internal network, interface to DMZ.** The AI model performing inference. It receives data that originated outside, which is why prompt injection reaches it.
4. **Internal network, protected.** User data and logs. Information disclosure and repudiation concentrate here.
5. **Internal network, development.** Training data and the training pipeline. Tampering concentrates here, and note that this zone writes into the inference model, so a compromise here propagates to production.

## What this architecture omits, and should not

The course diagram is a reasonable teaching baseline but is missing components that carry most of the real risk in a modern LLM deployment. When threat modeling a real system, add:

1. **RAG data sources and the vector store.** Retrieved documents are an input path to the model that bypasses the user entirely.
2. **Plugins, tools and agent actions.** Anything the model can call is a privilege the model holds, and therefore a privilege an injection inherits.
3. **The system prompt.** An asset in its own right, and one that leaks.
4. **Third party model APIs.** If inference is hosted elsewhere, the trust boundary moves outside the organisation.
5. **The model artifact and its registry.** Where weights are stored, and how they are loaded.

Compare the OWASP 2025 example architecture, which explicitly includes vector databases, RAG private data stores, server side functions, grounding and URL scraping from untrusted media, and labels three separate trust boundaries. That is much closer to what a production system looks like.
