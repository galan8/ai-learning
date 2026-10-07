# Diagramming for Threat Modeling

You cannot reason about what you cannot see. The diagram is the representation the Manifesto's definition refers to, and it is what the four questions are asked against.

## Why the data flow diagram

A **data flow diagram (DFD)** represents the flow of data through a process or system. It shows the inputs and outputs of each entity and of the process itself.

Crucially, **a DFD has no control flow: no decision rules, no loops, no branching.** That is a deliberate simplification, not an omission. Operations based on the data can be shown in a flowchart if needed; the DFD's job is only to show how data passes from one element to another.

DFDs are the standard for threat modeling precisely because of that simplicity. Attacks follow data. Where data crosses from one level of trust to another is where threats live, and stripping out control logic makes those crossings visible.

## The five elements

| Element | Notation | Meaning |
| ------- | -------- | ------- |
| **External entity / actor** | Rectangle | A third party or actor outside your control. Provides input to, or takes output from, the system |
| **Process / application** | Circle | Something that transforms data: an application, service or function |
| **Data store** | Two parallel horizontal lines (a rectangle without vertical edges) | A file or database: SQL, NoSQL, object store, vector store |
| **Data flow** | Arrow | Connects processes to each other, to data stores and to entities, showing direction. Request and response may be two arrows or one double headed arrow |
| **Trust boundary** | Dashed or solid enclosing line | A boundary where the level of trust changes |

Two conventions worth knowing: a **double circle** conventionally denotes a complex process that should be decomposed further, and **data stores connect to processes, not directly to external entities.**

## Levels

- **Level 0 (context diagram):** the whole system as a single process, with its external entities. Useful for scoping.
- **Level 1:** decomposes that into the major processes, data stores and flows. This is where most threat modeling happens.
- **Level 2 and beyond:** decomposes individual processes further, used only where the risk justifies the detail.

## Where threats come from

The trust boundary is the highest yield part of the diagram. Every flow that crosses one is a place where data arrives from somewhere less trusted, and therefore a place to apply STRIDE. A useful working rule: **if a data flow crosses a trust boundary, it deserves its own threat analysis.**

For AI systems, add two things most standard DFDs omit: the **training data flow** (which is a write path into the model's behaviour) and the **model artifact itself** (which is both an asset and, in pickle based formats, executable code). Microsoft's AI/ML threat modeling guidance explicitly recommends asking whether the system trains from user supplied input, because that single question changes the trust boundaries.
